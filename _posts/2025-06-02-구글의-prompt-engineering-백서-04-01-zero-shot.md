---
layout: post
title: 구글의 Prompt Engineering 백서 - "04-01 Zero-shot"
date: 2025-06-02 21:17 +0900
categories: [Prompt Engineering]
tags: [prompt, prompt-engineering, google-whitepaper]
toc: False
---

### **Summary**

Zero-shot은 예시가 주어지지 않고 task에 대한 설명과 필요한 최소한의 정보만 작성하는 Prompt 기법이다.<br>실습 1, 2를 통해 Zero-shot의 결과가 어떻게 나오는지 확인할 수 있다.

#### **Keywords**

`zero-shot`, `simplest type of prompt`, `no example`

### **Reference**

- [Google의 Prompt Engineering 백서](https://www.kaggle.com/whitepaper-prompt-engineering)
- [Structured Output](https://ai.google.dev/gemini-api/docs/structured-output?hl=ko)

---

Zero-shot은 가장 간단한 유형의 Prompt라고 할 수 있다. task에 대한 설명과 필요한 최소한의 정보가 LLM에 제공된다. 이러한 input은 질문, 이야기의 시작, instruction 등 무엇이든 가능하다.

Zero-shot은 "no example", 예시가 없음을 의미한다.
만약, Zero-shot Prompting으로 원하는 결과가 나오지 않았다면, One-shot 또는 Few-shot Prompting을 사용할 수 있다.

## **실습 1**

```python
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
    "max_output_tokens": 10,
}

contents = """Classify movie reviews as POSITIVE, NEUTRAL or NEGATIVE.
Review: 'Her' is a disturbing study revealing the direction humanity is headed \
if AI is allowed to keep evolving, unchecked. \
I wish there were more movies like this masterpiece.
Sentiment:"""

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
```

**실습 1** 코드를 실행하면 `POSITIVE` 값이 출력된다.<br>출력 제한으로 `max_output_tokens` 을 `10` 으로 제한했지만, `Enum` 을 사용하면 더 편리하게 출력값을 제한할 수 있다.

## **실습 2**

```python
from enum import Enum


class Sentiment(Enum):
    POSITIVE = "Positive"
    NEUTRAL = "Neutral"
    NEGATIVE = "Negative"


response = client.models.generate_content(
    model=CONFIGS["model"],
    config=types.GenerateContentConfig(
        temperature=CONFIGS["temperature"],
        top_p=CONFIGS["top_p"],
        response_mime_type="text/x.enum",
        response_schema=Sentiment,
    ),
    contents=contents,
)

print(response.text)
```

**실습 2** 코드를 실행하면 `Positive` 값이 나오게 되는데, 출력 길이의 수를 지정하지 않고도 Class로 원하는 값을 설정할 수 있다. 출력값을 강하게 제한할 때 유용하게 쓰인다. (`Enum` 외에도 [`JSON` 형식](https://ai.google.dev/gemini-api/docs/structured-output?hl=ko#generating-json)으로 생성하는 방식도 있다.)