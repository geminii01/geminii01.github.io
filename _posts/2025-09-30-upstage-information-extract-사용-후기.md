---
layout: post
title: Upstage Information Extract 사용 후기! (배달 주문 내역 정보 추출하기)
date: 2025-09-30 20:35 +0900
categories: [Upstage]
tags: [upstage-education, upstage-document-parse, solar-pro-2]
---

## **들어가며**

문서에서 **필요한 정보를 추출** 하는 작업은 생각보다 까다로운데요! 특히 PDF, 이미지, 스캔 문서 등 다양한 형식의 문서를 처리해야 할 때는 더욱 그렇습니다. 오늘은 Upstage의 **Information Extract** 을 사용해 보면서 이런 문제를 간단하게 해결할 수 있었던 경험을 공유하려 합니다.

이번 포스트에서는 배달 주문 내역 캡처 이미지에서 주문 정보를 자동으로 추출하는 실습을 통해 Information Extract의 기능과 장점을 소개하겠습니다!

### **References**

- [Upstage Information Extract Documentation](https://console.upstage.ai/docs/capabilities/extract/universal-extraction)

---

## **Upstage의 Information Extract란?**

### **Information Extract의 개념**

Upstage의 Information Extract는 **Universal information extraction** 기능을 제공하는 API입니다. 기존의 정보 추출 시스템은 특정 문서 유형에 맞춰 파인튜닝이 필요했지만, Information Extract는 **어떤 문서 유형이든 추가적인 학습 없이** 핵심 정보를 추출할 수 있습니다.

### **주요 특징**

**1/ Works with any document type**
- 복잡한 PDF, 스캔 이미지, Microsoft Office 문서 등 다양한 형식을 처리
- 문서 형식과 관계없이 원활한 데이터 추출 가능

**2/ Schema-agnostic processing**
- 사용자가 정의한 어떤 스키마든 동적으로 처리
- 사용 사례에 따라 즉석에서 커스터마이징 가능

**3/ Extracts hidden and implied information**
- 명시적으로 표시된 정보뿐만 아니라 함축되거나 추론 가능한 값도 추출
- 예: 여러 항목의 합계 계산, 직접 표시되지 않은 핵심 세부 정보 식별

**4/ No fine-tuning required**
- 추가적인 모델 학습이 없어도 필요한 정보를 추출

---

## **배달 주문 내역에서 정보 추출하기**

### **실습 목표**

배달 앱의 주문 내역 캡처 이미지에서 다음 정보를 자동으로 추출합니다:
- 주문 번호
- 주문 일시
- 매장 이름
- 총 주문 금액
- 배달비
- 총 할인 금액
- 주문한 메뉴 목록 (메뉴명, 수량, 가격)
- 결제 방법

### 구현 과정

```python
import os
import base64
import json

from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

# API Key 설정 (.env 파일에 Upstage API Key를 발급받아 저장)
UPSTAGE_API_KEY = os.environ["UPSTAGE_API_KEY"]

client = OpenAI(
    api_key=UPSTAGE_API_KEY,
    base_url="https://api.upstage.ai/v1/information-extraction",
)

def encode_img_to_base64(img_path):
    """이미지를 base64로 인코딩"""
    with open(img_path, "rb") as img_file:
        img_bytes = img_file.read()
        base64_data = base64.b64encode(img_bytes).decode("utf-8")
        return base64_data
```
[Upstage Console](https://console.upstage.ai/docs/getting-started)에서 API Key를 발급받아 `.env` 파일에 저장합니다.

---

```python
# 이미지 파일 읽기 및 base64 인코딩
filepath = "./Screenshot_xxxxxxxx_xxxxxx.jpg"
base64_data = encode_img_to_base64(filepath)
```
쿠팡이츠, 배달의 민족, 요기요 등 배달 앱의 주문 내역을 캡처해 주세요. \
코드 파일과 동일한 경로에 이미지 파일을 저장해 주면 됩니다!

저는 쿠팡이츠 주문 내역 이미지를 사용했는데요! \
주문 내역 탭 → 특정 주문의 영수증을 클릭하면 이미지를 확인할 수 있습니다!

---

```python
# Information Extract API 호출
extraction_response = client.chat.completions.create(
    model="information-extract",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:application/octet-stream;base64,{base64_data}"
                    },
                }
            ],
        }
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "document_schema",
            "schema": {
                "type": "object",
                "properties": {
                    # 주문 번호
                    "orderNumber": {
                        "type": "string",
                        "description": "Unique identifier for the order.",
                    },
                    # 주문 일시
                    "orderDate": {
                        "type": "string",
                        "description": "Date and time when the order was placed.",
                    },
                    # 매장 이름
                    "storeName": {
                        "type": "string",
                        "description": "Name of the store where the order was made.",
                    },
                    # 총 주문 금액
                    "totalAmount": {
                        "type": "number",
                        "description": "Total amount of the order.",
                    },
                    # 배달비
                    "deliveryFee": {
                        "type": "number",
                        "description": "Delivery fee for the order.",
                    },
                    # 총 할인 금액
                    "discountAmount": {
                        "type": "number",
                        "description": "Total discount applied to the order.",
                    },
                    # 주문한 메뉴 목록
                    "menuItems": {
                        "type": "array",
                        "description": "List of items in the order. Option writing is ignored.",
                        "items": {
                            "type": "object",
                            "properties": {
                                # 메뉴 이름
                                "itemName": {
                                    "type": "string",
                                    "description": "Name of the item.",
                                },
                                # 메뉴 수량
                                "quantity": {
                                    "type": "number",
                                    "description": "Quantity of the item.",
                                },
                                # 메뉴 가격
                                "price": {
                                    "type": "number",
                                    "description": "Price of the item.",
                                },
                            },
                        },
                    },
                    # 결제 방법
                    "paymentMethod": {
                        "type": "string",
                        "description": "Method of payment used for the order.",
                    },
                },
            },
        },
    },
)

# 결과 출력
print(json.dumps(json.loads(extraction_response.choices[0].message.content), indent=2, ensure_ascii=False))
```
```terminal
{
    "orderNumber": "0S6XXX",
    "orderDate": "2025-09-06 14:05",
    "storeName": "써브웨이 ABCD점",
    "totalAmount": 17500,
    "deliveryFee": 0,
    "discountAmount": 0,
    "menuItems": [
        {
            "itemName": "풀드포크 바비큐 (1 5cm)",
            "quantity": 1,
            "price": 8500
        },
        {
            "itemName": "스파이시 쉬림프 (1 5cm)",
            "quantity": 1,
            "price": 9000
        }
    ],
    "paymentMethod": "쿠페이 머니 결제, 쿠팡캐시 결제"
}
```
결과를 확인해 보면, 주문 번호, 주문 일시, 매장 이름, 총 주문 금액, 배달비, 총 할인 금액, 주문한 메뉴 목록, 결제 방법이 정확하게 추출되었습니다.

스키마를 작성할 때, 주문한 메뉴 목록의 이름과 수량, 가격을 추출하도록 했는데요. \
써브웨이를 주문할 때 여러 옵션도 선택했었는데, `menuItems` 의 `description` 에 **옵션은 무시하라고** 했기 때문에 추출되지 않았습니다.

자신이 정의하는 대로, 자신의 사례에 맞게 커스터마이징을 할 수 있는 점이 큰 장점이라고 생각합니다.😁

---

### **간단한 코드 설명**

**1/ 이미지 전처리**
- `encode_img_to_base64` 함수로 이미지를 base64 형식으로 변환
- API 요청 시 base64 인코딩된 이미지 데이터 전달

**2/ JSON 스키마 정의**
- `response_format` 에 추출하고자 하는 정보의 스키마를 JSON 형식으로 정의
- 각 필드에 타입과 설명(`description`)을 명시하여 추출 정확도 향상

**3/ API 호출**
- OpenAI SDK를 사용하여 Upstage Information Extract API 호출
- `model="information-extract"` 지정

---

## **Information Extract의 장점**

### **간편한 구현 , 유연한 스키마 정의 , 추가 학습 불필요**
복잡한 OCR 파이프라인이나 텍스트 파싱 로직을 직접 구현할 필요가 없습니다. 원하는 출력 형식을 JSON 스키마로 정의하기만 하면 됩니다. 

자신이 생각하는 대로 자연어로 작성할 수 있다는 게 인상적이지 않나요? \
문서 유형이 바뀌어도 스키마만 수정하면 되기 때문에, 송장, 계약서, 신분증 등 다양한 사용 사례에 적용할 수 있습니다.

이렇게 각 사례마다 파인튜닝이나 별도의 모델 학습 없이 바로 사용할 수 있어서, 개발 시간과 비용을 크게 절약할 수 있습니다!

---

## **활용 가능한 사례**

- **전자상거래**: 주문 내역, 배송 정보 자동 추출
- **회계/재무**: 영수증, 송장, 세금 문서 처리
- **법률/계약**: 계약서, 법률 문서의 핵심 조항 추출
- **인사/채용**: 이력서, 자기소개서에서 정보 추출
- **의료**: 검사 결과, 처방전, 진단서 처리
- **보험**: 청구서, 증빙 서류 분석

---

## **실전 프로젝트: 배달 주문 분석 시스템**

Information Extract를 활용하면 추출한 정보를 가지고 실용적인 서비스도 만들어 볼 수 있습니다!

저는 이 API를 사용해서 **배달 주문 내역을 자동으로 분석하는 시스템** 을 만들어봤는데요. 배달을 얼마나 시켰는지.. 확인해보고 싶었습니다.🥲

다음과 같은 기능을 구현했습니다:
- 📸 스크린샷을 업로드하여 Information Extract로 자동 정보 추출
- 🤖 Solar Pro 2를 활용한 음식 카테고리 자동 분류
- 📊 월별/카테고리별 소비 패턴 분석

제가 테스트해 본 결과, 배달의 민족이나 요기요의 주문 내역도 잘 추출했습니다. 스키마를 정의해놓으면 구조에 상관없이 올바르게 추출되어서 굉장히 좋았습니다.🫢

> 🔗 해당 코드는 [GitHub Repo](https://github.com/geminii01/delivery-hunters) 에서 확인하실 수 있습니다.
{: .prompt-info }

---

## **마치며**

Upstage의 Information Extract는 문서에서 정보를 추출하는 작업을 획기적으로 간소화합니다. 간단한 스키마 정의만으로 다양한 형식의 문서에서 정확하게 정보를 추출할 수 있다는 점이 가장 큰 장점입니다.

특히 파인튜닝이나 복잡한 전처리 과정 없이 바로 사용할 수 있어, 빠르게 프로토타입을 만들거나 실제 서비스에 적용하기에 적합하다고 생각합니다! 문서에서 특정 정보의 추출이 필요한 프로젝트가 있다면 Information Extract를 활용해 보시는 것을 추천드립니다!

---

#Upstage #DocumentIntelligence #SolarPro2 #UpstageInformationExtract