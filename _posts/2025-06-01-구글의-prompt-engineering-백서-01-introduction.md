---
layout: post
title: 구글의 Prompt Engineering 백서 - "01 Introduction"
date: 2025-06-01 23:37 +0900
categories: [Prompt Engineering]
tags: [prompt, prompt-engineering, google-whitepaper]
toc: False
---

### **Summary**
> You don’t need to be a data scientist or a machine learning engineer – everyone can write a prompt.

Prompt는 누구나 작성할 수 있지만, 최적의 Prompt를 작성하는 것은 어려운 일이다.<br>모델의 종류, Configurations, 단어, 톤, Context 등 여러 요소가 Prompt의 품질에 영향을 미치기 때문이다.

최적의 Prompt를 찾기 위해서는 이러한 요소들을 조정하고 평가하는 과정이 필요하다. 따라서 Prompt Engineering은 최적의 Prompt를 찾기 위한 **"반복적인(iterative) 프로세스"**라고 할 수 있다.

#### **Keywords**

`iterative process`

### **Reference**

- [Google의 Prompt Engineering 백서](https://www.kaggle.com/whitepaper-prompt-engineering)

---

Prompt는 LLM이 특정 Output(출력)을 예측하는 데 사용하는 Input(입력)이다. 여기서 "예측"이라는 단어가 나온 이유는 LLM은 학습 데이터를 기반으로, 다음에 올 Token을 예측하는 방식으로 동작하기 때문이다. LLM의 예측 과정을 자신이 원하는 방향으로 나올 수 있게끔 하는 작업을 Prompt Engineering이라고 할 수 있다.

이는 전문가만이 작성할 수 있는게 아니라, 누구나 작성할 수 있다는 것이 인상 깊다. 하지만 작성하다 보면, 최적의 Prompt를 작성하는 것은 어렵고 복잡한 일임을 깨닫는다.

그렇다면 작성할 때 무엇이 영향을 미치는걸까?<br>모델의 종류, 모델이 학습한 데이터, 모델의 Configurations(설정) 등이 있고, 단어 선택, 스타일과 톤(어조), 구조, Context 등 모든 것이 Prompt를 작성할 때 중요한 요소이다.

이러한 요소들이 서로 복잡하게 얽혀 있어서 최적의 Prompt를 찾기 위해선 다양한 시도를 해야 하기 때문에, **Prompt Engineering은 시행착오를 통해 최적의 Prompt를 찾는 Iterative한 과정**임을 알 수 있다.