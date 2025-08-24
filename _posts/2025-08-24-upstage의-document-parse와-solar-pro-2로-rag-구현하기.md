---
layout: post
title: Upstage의 Document Parse와 Solar Pro 2로 RAG 구현하기
date: 2025-08-24 05:16 +0900
categories: [Upstage]
tags: [tutorial, document-parse, solar-pro-2]
---

## **들어가며**

수십, 수백 페이지에 달하는 PDF 문서를 하나하나 넘겨보면서, 원하는 정보를 찾기 위해 헤맨 경험이 있으신가요? 이러한 검색 방식은 시간을 낭비할 뿐만 아니라, 필요한 정보는 놓치고 단편적인 내용에만 집중하게 될 수 있습니다.

본 튜토리얼에서는 **Document Parse**로 PDF 문서에서 텍스트를 추출하는 방법과 **Solar Pro 2**로 질문에 대해 정확한 답변을 생성하는 RAG 시스템을 함께 구현하면서, **Upstage**의 AI 모델들을 활용하는 것을 배울 수 있습니다.

### **RAG(Retrieval-Augmented Generation)란 무엇일까요?**

RAG는 LLM이 답변을 생성하기 전에, 먼저 외부 DB나 문서에서 관련 정보를 검색하여 그 내용을 기반으로 답변의 근거를 마련해 정확성과 신뢰도를 높이는 방식입니다. 이를 통해 LLM이 잘못된 정보를 생성하는 환각(Hallucination)을 크게 줄일 수 있습니다.

본 튜토리얼에서는 다음 3단계의 절차를 통해 RAG 시스템을 구현합니다.

1. **Indexing** : **Document Parse**를 사용해 PDF와 같은 문서를 불러와서 작은 단위의 청크로 나누고, 이를 벡터로 변환한 후 데이터베이스에 저장합니다.
2. **Retrieval** : 사용자 질문과 가장 유사한 문서 청크들을 검색합니다.
3. **Generation** : 검색된 정보를 참고 자료로 활용하여, 사용자 질문과 함께 **Solar Pro 2**에 전달하여 답변을 생성합니다.

### **References**

- [Build a Retrieval Augmented Generation (RAG) App: Part 1](https://python.langchain.com/docs/tutorials/rag/)
- [What is RAG (Retrieval-Augmented Generation)?](https://aws.amazon.com/what-is/retrieval-augmented-generation/)
- [Upstage Docs: Document Parse](https://console.upstage.ai/docs/capabilities/document-digitization/document-parsing)
- [복잡한 문서를 지능형 데이터로, 강력한 문서 파싱 기술 - 업스테이지 도큐먼트 파스(Document Parse)](https://www.upstage.ai/blog/ko/introduce-upstage-document-parse?page_slug=ko%2Fintroduce-upstage-document-parse)
- [Upstage Docs: Reasoning](https://console.upstage.ai/docs/capabilities/reasoning)

---

## **환경 설정**

> **💡 전체 코드 다운로드**
>
> 튜토리얼을 시작하기 전에, 실습에 필요한 모든 코드는 아래 GitHub Repository에서 다운로드하거나 복제(Clone)할 수 있습니다. Repository에는 본 튜토리얼의 전체 코드, 샘플 데이터(.pdf), 그리고 .env 파일 예시가 포함되어 있습니다.
>
> **[🔗 GitHub Repository](https://github.com/geminii01/upstage-rag-tutorial)**


1. 먼저 필요한 라이브러리들을 설치해 주세요.
    ```terminal
    pip install -r requirements.txt
    ```

2. Upstage API Key 발급이 필요합니다. [Upstage Console](https://console.upstage.ai/api-keys) 에서 API Key를 발급해 주세요.<br>발급받은 키를 안전하게 보관하기 위해, 프로젝트 폴더에 `.env` 라는 파일을 만들고 다음과 같이 내용을 작성해 주세요. `UPSTAGE_API_KEY="여기에 발급받은 API 키를 붙여넣으세요"`
    ```python
    import os
    import json
    import requests
    import warnings

    warnings.filterwarnings("ignore")

    from collections import Counter
    from dotenv import load_dotenv

    from openai import OpenAI
    from langchain_text_splitters import RecursiveCharacterTextSplitter
    from sentence_transformers import SentenceTransformer
    from qdrant_client import QdrantClient, models

    load_dotenv()
    api_key = os.getenv("UPSTAGE_API_KEY")
    ```

## **1. Indexing**

Indexing 단계는 원본 문서(예: PDF)를 검색 가능한 데이터베이스로 만드는 과정입니다. AI가 참고할 '오픈북'을 미리 잘 정리해두는 단계라고 생각할 수 있습니다. 전체 과정은 아래와 같이 Load → Split → Embedding → Store 순서로 진행됩니다.

- **Load** : PDF, 웹페이지 등 다양한 소스에서 데이터를 불러옵니다.
- **Split** : 긴 문서의 내용을 작은 의미 단위인 청크로 나눕니다.
- **Embedding** : 각 청크를 벡터로 변환합니다.
- **Store** : 벡터들을 검색 가능한 데이터베이스에 저장합니다.

## **Load**

가장 먼저, 원본 PDF 문서를 불러와 Document Parse로 분석하고, 불필요한 텍스트를 제거하는 전처리 과정을 거칩니다.

### **Document Parse로 PDF 파싱하기**

일반적인 PDF parser는 단순히 텍스트만 추출합니다.<br>하지만 Upstage의 Document Parse는 다음과 같은 특징이 있습니다:
- **구조 인식** : 제목, 본문, 표, 이미지, 목록 등을 정확히 구분해서 파싱
- **레이아웃 분석** : paragraph, heading1, header, figure, list 등 다양한 레이아웃 카테고리를 자동으로 분류
- **읽는 순서 보장** : 문서의 논리적 흐름에 따라 내용을 순서대로 정리
- **구조화된 변환** : 각 요소를 적절한 HTML 태그(`<h1>`, `<p>`, `<table>` 등)로 변환
- **높은 정확도** : 93.48% 문서 인식 정확도로 복잡한 문서도 안정적으로 처리

이런 특징 덕분에 단순한 텍스트 추출이 아닌, **문서의 의미와 구조를 이해**한 파싱이 가능합니다.<br>(Document Parse의 API 호출 시, 자세한 정보는 [이 링크](https://console.upstage.ai/api/document-digitization/document-parsing)를 참고해 주세요.)

<br>이번 튜토리얼에서는 저작권 문제가 없는 법령 문서를 사용하겠습니다. [국가법령정보센터](https://www.law.go.kr)에서 PDF를 다운받을 수 있습니다.
```python
# 사용할 문서를 작성
filename = "./data/전자서명법(법률)(제18479호)(20221020).pdf"

# API 호출
url = "https://api.upstage.ai/v1/document-digitization"
headers = {"Authorization": f"Bearer {api_key}"}
files = {"document": open(filename, "rb")}
data = {
    "output_formats": "['html', 'markdown', 'text']",  # 출력 형식을 지정할 수 있음
    "ocr": "force",
    "base64_encoding": "['figure']",
    "model": "document-parse",
}
dp_result = requests.post(url, headers=headers, files=files, data=data)

# 결과를 json 파일로 저장
with open("./dp_result.json", "w", encoding="utf-8") as f:
    json.dump(dp_result.json(), f, ensure_ascii=False, indent=2)
```

API 호출이 완료되어 json 파일에 파싱 결과가 저장되었습니다! 결과를 확인해 볼까요?
```python
# json 파일 로드
with open("./dp_result.json", "r", encoding="utf-8") as f:
    data = json.load(f)

# 파싱 결과 확인
categories = [elem["category"] for elem in data["elements"]]
print("[ Category 분포 확인 ]")
for category, count in Counter(categories).most_common():
    print(f"- {category}: {count}")

print("\n[ 파싱 결과 일부 확인 ]")
for elem in data["elements"][3:6]:
    print(f"ID {elem['id']}")
    print(f"- Category: {elem['category']}")
    print(f"- Content: {elem['content']['text'][:50]} ...")
    print(f"- Page: {elem['page']}\n")
```

OUTPUT:
```terminal
[ Category 분포 확인 ]
- paragraph: 42
- list: 18
- footer: 18
- header: 6
- heading1: 2

[ 파싱 결과 일부 확인 ]
ID 3
- Category: paragraph
- Content: 제1조(목적) 이 법은 전자문서의 안전성과 신뢰성을 확보하고 그 이용을 활성화하기 위하여  ...
- Page: 1

ID 4
- Category: heading1
- Content: 제2조(정의) 이 법에서 사용하는 용어의 뜻은 다음과 같다. ...
- Page: 1

ID 5
- Category: list
- Content: 1. "전자문서"란 정보처리시스템에 의하여 전자적 형태로 작성되어 송신 또는 수신되거나 저 ...
- Page: 1
```

#### **🤔 파싱된 결과는 어떤 구조를 가지고 있나요?**

파싱 결과는 크게 `content`와 `elements`로 구분됩니다.

`content`에서는 `html`, `markdown`, `text`와 같이 전체 결과를 한 번에 볼 수 있습니다.

`elements`에서는 **레이아웃별 정보**를 확인할 수 있습니다.
- `category` : 레이아웃 카테고리의 종류 (paragraph, heading1, list 등)
- `content` : 실제 텍스트 내용 (html, markdown, text 형식으로 제공)
- `coordinates` : 위치 좌표 정보 (x, y 값)
- `id` : 순서 번호 (0부터 시작)
- `page` : 페이지 번호 (1부터 시작)

```json
{
  "api": "2.0",
  "content": {
    "html": "<header id='0' style='font-size:20px'>전자서명법</header>\n<p id='1' data-category='paragraph' style='font-size:22px'>전자서명법</p>\n<br><p id='2' data-category='paragraph' style='font-size:16px'>[시행 2022. 10. 20.] [법률 제18479호, 2021. 10. 19., 일부개정]<br>...",
    "markdown": "전자서명법\n\n전자서명법\n\n[시행 2022. 10. 20.] [법률 제18479호, 2021. 10. 19., 일부개정]\n...",
    "text": "전자서명법\n전자서명법\n[시행 2022. 10. 20.] [법률 제18479호, 2021. 10. 19., 일부개정]\n..."
  },
  "elements": [
    {
      "category": "header",
      "content": {
        "html": "<header id='0' style='font-size:20px'>전자서명법</header>",
        "markdown": "전자서명법",
        "text": "전자서명법"
      },
      "coordinates": [
        {
          "x": 0.8624,
          "y": 0.0344
        },
        {
          "x": 0.9511,
          "y": 0.0344
        },
        {
          "x": 0.9511,
          "y": 0.0504
        },
        {
          "x": 0.8624,
          "y": 0.0504
        }
      ],
      "id": 0,
      "page": 1
    },
  ],
}
```

### **데이터 전처리**

파싱된 데이터에는 필수적인 내용 외에도 불필요한 요소들이 있습니다. RAG의 정확도를 높이기 위해 간단한 전처리를 진행해보겠습니다.

<br>먼저, 어떤 카테고리들이 불필요한 내용을 담고 있는지 확인해 봅시다.
```python
page_num = 0
category_name = "footer"  # paragraph, heading1, list, header, footer
print(f"[ 문서 구조 확인: {category_name} ]")
for elem in data["elements"]:
    if elem["page"] != page_num:
        page_num = elem["page"]
        print(f"\n=========== Page {page_num} ==========\n")

    if elem["category"] == category_name:
        print(elem["content"]["text"])
```

내용을 확인한 결과, `header`와 `footer`는 각각 페이지 제목과 쪽수로 파악되어 제거하겠습니다.
```python
texts = []
for elem in data["elements"]:
    category = elem["category"]
    text = elem["content"]["text"]
    # 필수 내용이 담긴 카테고리만 선택
    if category in ["paragraph", "list", "heading1"]:
        texts.append(text)
texts = "\n".join(texts)

print(f"- 전처리 전 텍스트 길이: {len(data["content"]["text"])}")
print(f"- 전처리 후 텍스트 길이: {len(texts)}")
print(f"\n- 첫 200자:\n{texts[:200]} ...")
```

OUTPUT:
```terminal
- 전처리 전 텍스트 길이: 8796
- 전처리 후 텍스트 길이: 8669

- 첫 200자:
전자서명법
[시행 2022. 10. 20.] [법률 제18479호, 2021. 10. 19., 일부개정]
과학기술정보통신부 (정보보호기획과) 044-202-6445, 6447
제1조(목적) 이 법은 전자문서의 안전성과 신뢰성을 확보하고 그 이용을 활성화하기 위하여 전자서명에 관한 기본적인
사항을 정함으로써 국가와 사회의 정보화를 촉진하고 국민생활의 편익을  ...
```

법령 문서의 앞부분에는 목차나 기타 정보들이 있어서, 실제 조문의 시작은 "제1조(목적)"부터 시작합니다. 추가적으로, "제1조(목적)"의 이전 부분을 제거하도록 하겠습니다.
```python
# 제1조(목적) 이전 부분 제거
start_pos = texts.find("제1조(목적)")
if start_pos != -1:
    cleaned_texts = texts[start_pos:]
else:
    cleaned_texts = texts
print(f"- 첫 200자:\n{cleaned_texts[:200]} ...")
```

OUTPUT:
```terminal
- 첫 200자:
제1조(목적) 이 법은 전자문서의 안전성과 신뢰성을 확보하고 그 이용을 활성화하기 위하여 전자서명에 관한 기본적인
사항을 정함으로써 국가와 사회의 정보화를 촉진하고 국민생활의 편익을 증진함을 목적으로 한다.
제2조(정의) 이 법에서 사용하는 용어의 뜻은 다음과 같다.
1. "전자문서"란 정보처리시스템에 의하여 전자적 형태로 작성되어 송신 또는 수신되거나 저 ...
```

출력 결과를 확인해보면, 본문의 이전 내용이 제거가 된 것을 알 수 있습니다.

## **Split**

Load 단계에서 정제된 긴 텍스트를, 한 번에 처리하기 좋은 작은 의미 단위인 **청크(Chunk)**로 분할하는 단계입니다.

#### **🤔 왜 청킹이 필요할까요?**
- **검색 정확도** : 작은 단위로 나누어 관련성 높은 내용을 정확히 찾도록 함
- **처리 효율성** : 임베딩 모델과 LLM의 입력 길이 제한 고려

<br>전처리가 완료된 텍스트 데이터를 작은 단위로 나누어 Embedding하기 좋은 형태로 만들어보겠습니다.
```python
# 문서를 1000자 단위로 분할
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,  # 각 청크의 최대 크기
    chunk_overlap=200,  # 청크 간 겹치는 부분 (맥락 유지)
)
split_texts = text_splitter.split_text(cleaned_texts)
print(f"총 {len(split_texts)}개의 청크로 분할 완료")
```

OUTPUT:
```terminal
총 11개의 청크로 분할 완료
```

LangChain의 `RecursiveCharacterTextSplitter`는 문단, 줄바꿈 등을 기준으로 텍스트를 나누는 TextSplitter입니다. 글자 수로만 나누는 것보다 문맥 유지에 유리합니다.

<br>다음으로, 분할된 각 청크에 법령 정보를 추가하여 문서를 구분할 수 있도록 합니다.
```python
documents = []
for content in split_texts:
    document = {
        "content": content,
        "metadata": {
            "filename": "전자서명법",
            "enforcement_date": "2022-10-20",
            "law_id": 18479,
        },
    }
    documents.append(document)

print("[ 첫번째 Document 확인 ]\n")
print(f'- Content:\n{documents[0]["content"][:200]} ...')
print(
    f'\n- Metadata:\n{json.dumps(documents[0]["metadata"], ensure_ascii=False, indent=2)}'
)
```

OUTPUT:
```terminal
[ 첫번째 Document 확인 ]

- Content:
제1조(목적) 이 법은 전자문서의 안전성과 신뢰성을 확보하고 그 이용을 활성화하기 위하여 전자서명에 관한 기본적인
사항을 정함으로써 국가와 사회의 정보화를 촉진하고 국민생활의 편익을 증진함을 목적으로 한다.
제2조(정의) 이 법에서 사용하는 용어의 뜻은 다음과 같다.
1. "전자문서"란 정보처리시스템에 의하여 전자적 형태로 작성되어 송신 또는 수신되거나 저 ...

- Metadata:
{
  "filename": "전자서명법",
  "enforcement_date": "2022-10-20",
  "law_id": 18479
}
```

청킹된 문서와 함께, metadata도 확인할 수 있습니다. 이 외에도 원하는 정보를 편하게 추가할 수 있습니다.

## **Embedding & Store**

분할된 청크들을 컴퓨터가 의미를 이해할 수 있는 **숫자 벡터로 변환(Embedding)**하고, 나중에 빠르게 검색할 수 있도록 **벡터 데이터베이스에 저장(Store)**합니다.

#### **🤔 텍스트를 벡터로 변환**

컴퓨터가 텍스트의 의미를 이해할 수 있도록 숫자 벡터로 변환하는 과정입니다. 비슷한 의미의 텍스트들은 비슷한 벡터값을 가지게 됩니다.
```python
# 임베딩 모델 불러오기
embedding_model = SentenceTransformer("Qwen/Qwen3-Embedding-4B")
```

<br>이제, 벡터들을 빠르게 검색할 수 있는 Qdrant 벡터 데이터베이스를 구축하겠습니다.<br>(Qdrant에 대한 설명은 [이 링크](https://qdrant.tech/documentation/overview/)를 참고해 주세요.)
```python
qdrant_path = "./qdrant"  # 로컬에 Qdrant 결과를 저장
collection_name = "law"  # 문서가 저장될 그룹의 이름

client = QdrantClient(path=qdrant_path)

# collection 생성
client.create_collection(
    collection_name=collection_name,
    vectors_config=models.VectorParams(
        size=embedding_model.get_sentence_embedding_dimension(),
        distance=models.Distance.COSINE,  # 코사인 유사도 사용
    ),
)
```

문서가 저장될 그룹이 생겼으니, 작은 단위로 분할된 청크들을 벡터로 변환하여 데이터베이스에 저장해 보겠습니다.
```python
all_points = []
for point_id, doc in enumerate(documents):
    # 텍스트를 벡터로 변환
    vector = embedding_model.encode(doc["content"]).tolist()

    # Qdrant에 저장할 point 생성
    point = models.PointStruct(
        id=point_id,
        vector=vector,  # 변환된 벡터
        payload={**doc},  # 벡터로 변환되지 않은 원본 텍스트와 메타데이터를 저장
    )
    all_points.append(point)

# 모든 벡터를 Qdrant에 업로드
client.upload_points(
    collection_name=collection_name,
    points=all_points,
)
```

Embedding과 Store의 과정을 끝으로, Indexing 단계가 완료되었습니다! 이제 사용자 질문과 관련된 문서를 검색하러 가볼까요?

## **2. Retrieval**

사용자의 질문과 가장 유사한 문서 청크들을 찾는 단계입니다.<br>질문도 앞서 사용한 동일한 임베딩 모델로 벡터화한 후, 저장된 문서들과의 유사도를 비교하여 관련성이 높은 내용을 찾아보겠습니다.

<br>전자서명의 발전을 위한 시책에 대해 질문하는 것을 예시로 들겠습니다.
```python
user_query = "전자서명의 발전을 위한 시책에는 어떤 것들이 있어?"

# 질문을 벡터로 변환
query_vector = embedding_model.encode(user_query).tolist()
```

이제, 질문과 가장 유사한 상위 3개의 문서 청크를 찾아보겠습니다.
```python
hits = client.search(
    collection_name=collection_name,
    query_vector=query_vector,
    limit=3,  # 상위 3개로 설정
)

print("[ 검색 결과 (상위 3개) ]")
for i, hit in enumerate(hits):
    print(f"\n{i}번째 결과 (유사도: {hit.score:.4f})")
    print(f"Content: {hit.payload['content'][:250]}...")
    print("-" * 50)
```

OUTPUT:
```terminal
[ 검색 결과 (상위 3개) ]

0번째 결과 (유사도: 0.7321)
Content: 제3조(전자서명의 효력) ① 전자서명은 전자적 형태라는 이유만으로 서명, 서명날인 또는 기명날인으로서의 효력이 부
인되지 아니한다.
② 법령의 규정 또는 당사자 간의 약정에 따라 서명, 서명날인 또는 기명날인의 방식으로 전자서명을 선택한 경우
그 전자서명은 서명, 서명날인 또는 기명날인으로서의 효력을 가진다.
제4조(전자서명의 발전을 위한 시책 수립) 정부는 전자서명의 안전성, 신뢰성 및 전자서명수단의 다양성을 확보하고 그
이용을 활성화하는 등...
--------------------------------------------------

1번째 결과 (유사도: 0.5466)
Content: 6. 그 밖에 전자서명의 이용 촉진을 위하여 필요한 사항
제6조(다양한 전자서명수단의 이용 활성화) ① 국가는 생체인증, 블록체인 등 다양한 전자서명수단의 이용 활성화를 위
하여 노력하여야 한다.
② 국가는 법률, 국회규칙, 대법원규칙, 헌법재판소규칙, 중앙선거관리위원회규칙, 대통령령 또는 감사원규칙에서
전자서명수단을 특정한 경우를 제외하고는 특정한 전자서명수단만을 이용하도록 제한하여서는 아니 된다.
제7조(전자서명인증업무 운영기준 등) ① 과...
--------------------------------------------------

2번째 결과 (유사도: 0.5346)
Content: 제1조(목적) 이 법은 전자문서의 안전성과 신뢰성을 확보하고 그 이용을 활성화하기 위하여 전자서명에 관한 기본적인
사항을 정함으로써 국가와 사회의 정보화를 촉진하고 국민생활의 편익을 증진함을 목적으로 한다.
제2조(정의) 이 법에서 사용하는 용어의 뜻은 다음과 같다.
1. "전자문서"란 정보처리시스템에 의하여 전자적 형태로 작성되어 송신 또는 수신되거나 저장된 정보를 말한다.
2. "전자서명"이란 다음 각 목의 사항을 나타내는 데 이용하기 위하...
--------------------------------------------------
```

유사도가 약 0.7로 가장 높은 점수가 나온 0번째 결과에 질문과 관련된 정보가 있는 것을 알 수 있습니다.

## **3. Generation**

검색된 문서를 바탕으로 **Solar Pro 2** 가 자연스러운 한국어 답변을 생성하는 단계입니다.

<br>LLM이 정확한 답변을 생성할 수 있도록, 검색된 정보와 사용자 질문을 적절히 조합한 프롬프트를 만들어야 합니다.
```python
prompt_template = """사용자는 법률 문서와 관련된 질문을 하고 있습니다. \
검색된 정보를 기반으로, 질문에 **간결하게** 답변해 주세요.

RETRIEVED INFORMATION:
{retrieved_documents}

USER QUESTION:
{user_query}"""


def format_docs(hits):
    """검색된 문서들을 하나의 문자열로 합치기"""
    doc_list = [hit.payload["content"] for hit in hits]
    return "\n\n---\n\n".join(doc_list)


# 최종 프롬프트 확인
final_prompt = prompt_template.format(
    retrieved_documents=format_docs(hits),
    user_query=user_query,
)
print("[ 생성된 Prompt 미리보기 ]")
print(final_prompt[:300], "...")
```

OUTPUT:
```terminal
[ 생성된 Prompt 미리보기 ]
사용자는 법률 문서와 관련된 질문을 하고 있습니다. 검색된 정보를 기반으로, 질문에 **간결하게** 답변해 주세요.

RETRIEVED INFORMATION:
제3조(전자서명의 효력) ① 전자서명은 전자적 형태라는 이유만으로 서명, 서명날인 또는 기명날인으로서의 효력이 부
인되지 아니한다.
② 법령의 규정 또는 당사자 간의 약정에 따라 서명, 서명날인 또는 기명날인의 방식으로 전자서명을 선택한 경우
그 전자서명은 서명, 서명날인 또는 기명날인으로서의 효력을 가진다.
제4조(전자서명의 발전을 위한 시책 수립) 정부는 전자서명의 안전성 ...
```

최종 프롬프트가 완성되었습니다. 드디어 **Solar Pro 2**로 답변을 생성할 차례입니다.
```python
client = OpenAI(
    api_key=api_key,
    base_url="https://api.upstage.ai/v1",
)

# 답변 생성
stream = client.chat.completions.create(
    model="solar-pro2",
    messages=[{"role": "user", "content": final_prompt}],
    reasoning_effort="high",  # ⭐️ 추론 모드
    stream=True,
    temperature=0.6,
    top_p=0.9,
    max_tokens=1024,
    # 이 외에도 다른 파라미터를 설정할 수 있습니다
)

# stream 출력
for chunk in stream:
    response = chunk.choices[0].delta.content
    if response is not None:
        print(response, end="")
```

OUTPUT:
```terminal
전자서명의 발전을 위한 시책은 **제4조**에 따라 다음과 같이 수립·시행됩니다:  

1. **신뢰성 제고 및 서명수단 다양성 확보**  
2. **전자서명 제도 개선 및 법령 정비**  
3. **가입자·이용자 권익 보호**  
4. **전자서명 상호연동 촉진**  
5. **기술 개발, 표준화, 인력 양성**  
6. **안전한 암호 사용을 통한 신뢰성 확보**  
7. **외국 전자서명 상호인정 등 국제협력**  
8. **공공서비스 전자서명 안전 관리**  
9. **그 외 발전에 필요한 사항**  

이 시책은 정부가 전자서명의 안전성, 신뢰성, 이용 활성화를 목표로 추진합니다. (출처: 제4조)
```

사용자의 질문에 근거가 있는 정확한 답변이 생성되었습니다!

## **Summary**

이번 튜토리얼을 통해, **Upstage**의 주요 모델들을 활용하여 RAG 시스템을 구현해보았습니다.

먼저 Indexing 단계에서는 **Document Parse**를 사용해 PDF를 구조화된 데이터로 변환했습니다. Retrieval 단계에서는 사용자 질문과 가장 유사한 문서 청크들을 검색했습니다. 마지막 Generation 단계에서는 검색된 정보를 근거로 하여 **Solar Pro 2**가 정확한 답변을 생성하는 과정을 확인했습니다.

### **Next Step**

Upstage의 Document Parse와 Solar Pro 2의 사용 경험은 어떠셨나요? 배운 내용을 바탕으로 다음 단계를 진행해 보세요!
- 다양한 문서로 확장하기 : 법령 문서 외에 매뉴얼, 보고서, 논문 등 자신만의 문서로 테스트해보세요.
- 프롬프트 개선하기 : 더 구체적이거나 도메인에 특화된 프롬프트를 작성하여 답변의 품질을 높여보세요.
- 나만의 챗봇 서비스 만들기 : 오늘 만든 RAG 시스템을 기반으로 Streamlit을 활용하여 챗봇 서비스를 구현해보세요.