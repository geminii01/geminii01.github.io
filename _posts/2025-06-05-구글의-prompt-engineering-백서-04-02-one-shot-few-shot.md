---
layout: post
title: 구글의 Prompt Engineering 백서 - "04-02 One-shot & Few-shot"
date: 2025-06-05 22:17 +0900
categories: [Prompt Engineering]
tags: [prompt, prompt-engineering, google-whitepaper]
toc: False
---

### **Summary**
> Examples are especially useful when you want to steer the model to a certain output structure or pattern.

Prompt에 예시를 포함하는 One-shot과 Few-shot 방식은 모델이 특정 structure나 pattern을 따르도록 유도한다. Few-shot은 여러 예시를 통해 모델이 일관된 패턴을 따를 가능성을 높이며, 일반적으로 3-5개의 예시를 사용하지만 task의 복잡성과 모델의 성능에 따라 조절이 필요하다. 강건한(Robust) 결과를 얻기 위해서는 task와 관련성이 높고, 다양하며, Edge case까지 포함된 고품질의 예시를 사용하는 것이 중요하다.

실습 1부터 4까지 Few-shot 기법의 적용과 구조화된 출력 생성의 과정을 단계별로 확인할 수 있다.

#### **Keywords**

`one-shot`, `few-shot`, `single example`, `multiple example`, `robust`, `edge case`

### **Reference**

- [Google의 Prompt Engineering 백서](https://www.kaggle.com/whitepaper-prompt-engineering)
- [Schema 구성](https://ai.google.dev/gemini-api/docs/structured-output?hl=ko#configuring-a-schema)
- [Enum 값 생성](https://ai.google.dev/gemini-api/docs/structured-output?hl=ko#generating-enums)

---

Prompt에 예시(Example)를 포함하면, LLM이 output을 특정 구조(Structure)나 패턴(Pattern)에 맞추어 생성하도록 유도할 수 있다. 예시의 수에 따라 One-shot, Few shot으로 구분된다.

One-shot은 단일 예시(single example)를 제공하는 것으로, 모델이 이 예시를 모방(imitate)하여 주어진 task를 어떻게 완료할 지 알 수 있다.<br>반면, Few-shot은 여러 예시(multiple example)를 제공하는 것이다. 이는 모델에게 일관된 규칙(pattern)을 알려준다. 하나의 예시를 모방하는 것을 넘어, 여러 예시 속에서 공통된 pattern을 파악하고 따르도록 할 수 있다.

Few-shot에서 필요한 예시의 수는 task의 복잡성(complexity), 예시의 품질(quality), 사용 중인 모델의 성능에 따라 달라진다. 일반적으로 최소 3-5개의 예시를 사용하는데, 복잡성이 높은 task에는 이보다 더 많은 예시가 필요하거나, 모델의 입력 길이 제한(input length limitation)으로 더 적은 예시를 사용해야 할 수 있다.

예시의 내용을 작성할 땐 task와 관련된 것들로 구성해야 하며, 예시는 다양하고 높은 품질로 잘 작성되어야 한다. 작은 것 하나가 모델을 혼란스럽게(confuse) 하여 원하는 결과를 얻지 못할 수 있다.

다양한 케이스의 입력에 대해 강건한(robust) 출력을 얻고자 하는 경우, 예시에 Edge case를 포함하는 것이 좋다. Edge case는 일반적이지 않고 예상 범위를 벗어나는 Input이라고 할 수 있다. Prompt를 작성할 때 이러한 Edge case를 고려하여 모델이 적절히 대응하도록 유도하는 것이 중요하다.

## **실습 1**
> JSON 형식으로 피자 주문을 추출

````python
# pip install google-genai python-dotenv
# .env 파일을 미리 생성해야 함

import os

from google import genai
from google.genai import types
from dotenv import load_dotenv

load_dotenv()


GOOGLE_API_KEY = os.environ["GOOGLE_API_KEY"]

CONFIGS = {
    "model": "gemini-2.0-flash",
    "temperature": 0.1,
    "top_p": 0.95,
    "max_output_tokens": 250,
}

contents = """Parse a customer's pizza order into valid JSON:

EXAMPLE:
I want a small pizza with cheese, tomato sauce, and pepperoni.

JSON Response:
```
{
"size": "small",
"type": "normal",
"ingredients": [["cheese", "tomato sauce", "peperoni"]]
}
```

EXAMPLE:
Can I get a large pizza with tomato sauce, basil and mozzarella.

JSON Response:
```
{
"size": "small",
"type": "normal",
"ingredients": [["tomato sauce", "basil", "mozzarella"]]
}
```

Now, I would like a large pizza, with the first half cheese and mozzarella. \
And the other tomato sauce, ham and pineapple.

JSON Response:"""


client = genai.Client(api_key=GOOGLE_API_KEY)
response = client.models.generate_content(
    model=CONFIGS["model"],
    config=types.GenerateContentConfig(
        temperature=CONFIGS["temperature"],
        top_p=CONFIGS["top_p"],
        max_output_tokens=CONFIGS["max_output_tokens"],
    ),
    contents=contents,
)

print(response.text)
````


```terminal
{
"size": "large",
"type": "half_and_half",
"ingredients": [
    ["cheese", "mozzarella"],
    ["tomato sauce", "ham", "pineapple"]
]
}
```

피자 주문을 받을 때 사이즈와 유형 그리고 재료를 추출해 보았다. 예시에는 반반이 아니라 한 판으로 된 예시만 있는데도, 여러 예시를 통해 pattern을 잘 따른 것을 알 수 있다. (모델의 좋은 성능 덕분에 원하는 결과가 바로 나온 것일 수 있다.)

반반에 대한 예시를 Prompt에 추가적으로 넣어주거나, 모델에 Schema를 제공할 수 있다.

## **실습 2**
> Prompt에 반반 예시를 추가

````python
contents_2 = """Parse a customer's pizza order into valid JSON:

...{생략}...

EXAMPLE:
I would like a medium pizza, half with mushrooms and cheese, half with pepperoni and olives.

JSON Response:
```
{
"size": "medium",
"type": "half_half",
"ingredients": [["mushrooms", "cheese"], ["pepperoni", "olives"]]
}
```

Now, I would like a small pizza, with the first half cheese and mozzarella. \
And the other tomato sauce, ham and pineapple.

JSON Response:"""


response = client.models.generate_content(
    model=CONFIGS["model"],
    config=types.GenerateContentConfig(
        temperature=CONFIGS["temperature"],
        top_p=CONFIGS["top_p"],
        max_output_tokens=CONFIGS["max_output_tokens"],
    ),
    contents=contents_2,
)

print(response.text)
````

```terminal
{
"size": "small",
"type": "half_half",
"ingredients": [["cheese", "mozzarella"], ["tomato sauce", "ham", "pineapple"]]
}
```

`contents` 와 달리 `contents_2` 에는 반반에 대한 예시를 추가로 작성하여, "type"에 `half_half` 라고 잘 생성된 것을 알 수 있다.

다음으로, JSON Schema를 정의해주면 모델이 구조화된 출력을 쉽게 생성하도록 유도할 수 있다.

## **실습 3**
> 모델에 Schema를 제공하여 출력 형식을 제한

```python
from pydantic import BaseModel


class PizzaOrder(BaseModel):
    SIZE: str
    TYPE: str
    INGREDIENTS: list[list[str]]


contents_3 = """Please parse the customer's pizza order below and extract the relevant information.

Order: I would like a small pizza, with the first half cheese and mozzarella. \
And the other tomato sauce, ham and pineapple."""

response = client.models.generate_content(
    model=CONFIGS["model"],
    config=types.GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=PizzaOrder,
    ),
    contents=contents_3,
)

print(response.text)
```

```terminal
{
  "SIZE": "small",
  "TYPE": "pizza",
  "INGREDIENTS": [
    ["cheese", "mozzarella"],
    ["tomato sauce", "ham", "pineapple"]
  ]
}
```

`contents_3` 에서는 이전 실습들과 다르게 Prompt에 예시를 포함하지 않고, `PizzaOrder` 을 정의하여 `response_schema` 로 전달했다. 이 방식은 모델에게 **명시적으로 출력 구조를 알려주어** 원하는 JSON 형식을 안정적으로 얻을 수 있다.

결과를 보면, 정의한 대로 `SIZE`, `TYPE`, `INGREDIENTS` 의 Key값을 가진 JSON이 정확하게 생성되었다.

다만, `TYPE` 의 값이 "pizza"로 나왔는데, `contents_3` 은 "고객의 피자 주문을 분석하고 관련 정보를 추출"하라는 일반적인 지시만 하고 있고, `TYPE` 에 어떤 종류의 값("normal", "half_half")이 들어가야 하는지에 대한 세부 지침이 없었다. Schema는 구조는 제한할 수 있지만, 각 필드에 들어갈 값의 내용(의미)까지 완벽하게 제어하지 못할 수 있다.

프롬프트에 해당 정보를 추론할 수 있는 설명을 추가하거나, `Enum` 을 사용하여 값을 제한하는 방법을 고려할 수 있다. (Schema로 출력 형식을 쉽게 제어할 수 있지만, Prompt의 내용도 여전히 중요한 것을 알 수 있다.)

## **실습 4**
> `Enum` 을 추가하여 정보를 추출

```python
from enum import Enum


class PizzaSize(Enum):
    SMALL = "small"
    MEDIUM = "medium"
    LARGE = "large"


class PizzaType(Enum):
    NORMAL = "normal"
    HALF_HALF = "half_half"


class PizzaOrder(BaseModel):
    SIZE: PizzaSize
    TYPE: PizzaType
    INGREDIENTS: list[list[str]]


response = client.models.generate_content(
    model=CONFIGS["model"],
    config=types.GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=PizzaOrder,
    ),
    contents=contents_3,
)

print(response.text)
```

```terminal
{
  "SIZE": "small",
  "TYPE": "half_half",
  "INGREDIENTS": [
    [
      "cheese",
      "mozzarella"
    ],
    [
      "tomato sauce",
      "ham",
      "pineapple"
    ]
  ]
}
```

`PizzaSize` 와 `PizzaType` 을 `PizzaOrder` 의 `SIZE` 및 `TYPE` 필드에 적용함으로써, 각 필드의 값의 범위를 명확히 지정하고 제한할 수 있었다.

**실습 3** 과 동일한 Prompt(`contents_3`)를 사용했음에도, 모델은 `SIZE` 를 `PizzaSize` 에 정의된 "small" 로, `TYPE` 을 `PizzaType` 에 명시된 "half_half"로 정확히 식별하여 출력했다.

이 실습으로 **실습 3** 에서의 문제를 해결할 수 있었다. 이처럼 JSON Schema 에서 `Enum` 을 같이 사용하면, 모델이 생성할 수 있는 값을 제한해주어, 답변의 일관성과 정확성을 향상시킬 수 있다. 이는 특정 카테고리나 정해져있는 값이 필요할 때 유용하다.