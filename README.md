# GlobalGates AI 발표 자료

> **피드로 여는 기업용 비즈니스 소셜 마켓**
>
> GlobalGates는 국내 중소기업과 해외 바이어를 연결하는 B2B 소셜 마켓입니다.
> 이 문서는 서비스 기획 배경을 데이터로 확인한 과정과, 그 결과를 바탕으로 구현한 AI 기능을 정리한 발표 자료입니다.

---

## 목차

1. [기획 배경: 데이터로 본 문제](#1-기획-배경-데이터로-본-문제)
2. [분석 결과에서 AI 기능으로](#2-분석-결과에서-ai-기능으로)
3. [상품 카테고리 분류](#3-상품-카테고리-분류)
4. [팔로우 추천](#4-팔로우-추천)
5. [무역 문서 RAG 챗봇](#5-무역-문서-rag-챗봇)
6. [n8n 뉴스 자동화](#6-n8n-뉴스-자동화)
7. [Spring Boot와 FastAPI 연동](#7-spring-boot와-fastapi-연동)
8. [최종 정리](#8-최종-정리)

---

## 1. 기획 배경: 데이터로 본 문제

### 분석 질문

GlobalGates의 출발점은 “중소기업이 해외에 진출하기 어렵다”는 막연한 주장으로 두지 않았습니다.
먼저 공개 통계 데이터를 보고 다음 질문을 확인했습니다.

| 질문 | 확인하려는 내용 |
|---|---|
| 국내 기업 중 중소기업 비중은 어느 정도인가? | 서비스 대상이 충분히 큰가 |
| 수출 교역액도 중소기업 중심인가? | 실제 거래 성과가 어디에 집중되어 있는가 |
| 중소기업 수출은 줄고 있는가, 성장하고 있는가? | 플랫폼이 붙을 만한 성장 흐름이 있는가 |

### 사용 데이터

| 데이터 | 출처 | 사용 목적 |
|---|---|---|
| 산업별·기업규모별 활동 기업 수 | 기업특성별무역통계 | 대기업/중소기업 활동 기업 수 비중 계산 |
| 기업규모별 수출입 통계 | 기업특성별무역통계 | 기업 규모별 수출 교역액 비중 계산 |
| 2015~2023 중소기업 수출 데이터 | 기업특성별무역통계 | 수출 참여 중소기업 수와 교역액 추이 확인 |

### 분석 과정 1: 기업 수와 수출 교역액 비교

먼저 기업 수 기준으로 중소기업이 얼마나 많은지 보고, 같은 기준에서 수출 교역액이 어떻게 분포하는지 비교했습니다.

```python
import pandas as pd

df1 = pd.read_excel(
    "./datasets/산업별_기업_규모별_기업수_활동_신생_소멸__20260504153638.xlsx",
    sheet_name="데이터_long_활동",
)

df2 = pd.read_excel(
    "./datasets/기업규모별_수출강도별_수출입_20260504152906.xlsx",
    sheet_name="tidy",
)
```

```python
# 2024년 산업 전체 기준 활동 기업 수 비중
count_share = count_slice.copy()
count_share["비중"] = count_share["활동기업수"] / count_share["활동기업수"].sum()
count_share["기업규모"] = pd.Categorical(
    count_share["기업규모"],
    ["대기업", "중소기업"],
)
count_share = count_share.sort_values("기업규모").reset_index(drop=True)
count_share
```

```python
# 2023년 수출 교역액 기준 대기업과 그 외 기업 비중
value_raw = value_slice.groupby("기업규모")["교역액(천달러)"].sum()

value_share = pd.DataFrame({
    "구분": ["대기업", "그 외"],
    "교역액(천달러)": [
        value_raw["대기업"],
        value_raw.drop("대기업").sum(),
    ],
})
value_share["비중"] = value_share["교역액(천달러)"] / value_share["교역액(천달러)"].sum()
value_share
```

분석 결과는 다음과 같았습니다.

| 항목 | 값 |
|---|---:|
| 2024년 활동 대기업 수 | 10,256개 |
| 2024년 활동 중소기업 수 | 7,631,495개 |
| 활동 기업 중 중소기업 비중 | 99.87% |
| 2023년 대기업 수출 교역액 | 407,719,203천달러 |
| 2023년 그 외 기업 수출 교역액 | 224,506,622천달러 |
| 대기업 수출 교역액 비중 | 64.49% |

![중소기업 수출 구조 분석](images/export_gap_analysis_01.png)

이 이미지는 “중소기업이 많다”와 “수출 교역액이 중소기업 중심이다”가 같은 말이 아니라는 점을 보여줍니다.
활동 기업 수는 거의 대부분 중소기업이지만, 수출 교역액은 대기업 쪽에 크게 몰려 있습니다.

### 분석 과정 2: 중소기업 수출 추이 확인

다음으로 중소기업 수출이 정체되어 있는지, 성장 흐름이 있는지 확인했습니다.

```python
sme_trend = (
    trend_slice
    .groupby("연도")[["기업수(개)", "교역액(천달러)"]]
    .sum()
    .reset_index()
)
sme_trend["기업수(개)"] = sme_trend["기업수(개)"].astype(int)
sme_trend
```

| 연도 | 수출 참여 중소기업 수 | 수출 교역액 |
|---:|---:|---:|
| 2015 | 88,225개 | 90,959,103천달러 |
| 2023 | 93,912개 | 104,826,969천달러 |
| 변화율 | +6.4% | +15.2% |

![중소기업 수출 추이](images/export_gap_analysis_02.png)

두 번째 이미지는 중소기업 수출이 완전히 정체된 시장이 아니라는 점을 보여줍니다.
2015년부터 2023년까지 수출 참여 중소기업 수와 교역액은 모두 증가했습니다.

### 기획 결론

분석 결과를 서비스 관점으로 정리하면 다음과 같습니다.

| 분석 결과 | 서비스 관점 |
|---|---|
| 중소기업은 기업 수 기준으로 압도적으로 많음 | 서비스 대상은 충분히 크다 |
| 수출 교역액은 대기업에 집중되어 있음 | 중소기업의 판로·노출·연결 문제가 남아 있다 |
| 중소기업 수출은 장기적으로 성장 중 | 도와야 할 대상이 아니라, 더 잘 연결될 수 있는 대상이다 |

따라서 GlobalGates의 문제 정의는 “중소기업은 수출을 못 한다”가 아닙니다.

> **수출 가능성이 있는 중소기업이 더 쉽게 상품을 노출하고, 거래 상대를 찾고, 무역 정보를 이해할 수 있는 구조가 필요하다.**

이 지점에서 AI 기능이 자연스럽게 필요해졌습니다.

---

## 2. 분석 결과에서 AI 기능으로

통계 분석은 서비스의 큰 방향을 잡기 위한 근거였습니다.
AI 기능은 그 방향을 실제 사용자의 행동 단위로 쪼개서 적용했습니다.

| 사용자 문제 | AI 기능 | 구현 결과 |
|---|---|---|
| 상품을 등록할 때 카테고리 선택이 번거로움 | 상품 카테고리 분류 | 상품명·설명·태그 기반 Top-3 추천 |
| 바이어/셀러 후보를 직접 찾아야 함 | 팔로우 추천 | bio·관심 카테고리 기반 회원 추천 |
| 무역 문서에서 필요한 내용을 찾기 어려움 | RAG 챗봇 | PDF 문서 근거 기반 답변 |
| 경제 뉴스를 계속 확인해야 함 | n8n 자동화 | RSS 수집 후 OpenAI 요약 저장 |

이번 AI 파트에서 중요한 점은 네 기능이 모두 실제 서비스와 연결되어 있다는 것입니다.
분류 모델은 상품 등록 화면에 연결되고, 추천 모델은 회원 추천 API로 제공되며, RAG는 챗봇 질의로 연결되고, n8n은 뉴스 요약 데이터를 DB에 저장합니다.

---

## 3. 상품 카테고리 분류

> **목표**: 상품명, 설명, 태그를 입력하면 상품 등록에 사용할 카테고리 후보 3개를 추천합니다.

### 데이터 구성

분류 모델은 네이버 쇼핑 데이터와 네이버 뉴스/블로그 데이터를 합쳐서 학습했습니다.

```python
shop_df = pd.read_csv("./datasets/naver_openapi_raw.csv")
news_df = pd.read_csv("./datasets/naver_news_blog_raw.csv")

g_df = pd.concat([shop_df, news_df])
g_df.reset_index(drop=True, inplace=True)
g_df.drop_duplicates(inplace=True, ignore_index=True)
```

| 데이터 | 규모 | 역할 |
|---|---:|---|
| 네이버 쇼핑 OpenAPI | 70,000건 | 상품명·카테고리 중심 학습 |
| 네이버 뉴스/블로그 OpenAPI | 112,000건 | 무역 행위·주제 텍스트 보강 |
| 최종 학습 데이터 | 176,426건 | 12개 카테고리 분류 학습 |
| CountVectorizer 어휘 수 | 396,276개 | 텍스트 벡터화 |

### 라벨 인코딩과 train/test 분리

```python
from sklearn.preprocessing import LabelEncoder
from sklearn.model_selection import train_test_split

category_encoder = LabelEncoder()
targets = category_encoder.fit_transform(g_df.category)
g_df["Target"] = targets

X_train, X_test, y_train, y_test = train_test_split(
    g_df.text,
    g_df.Target,
    stratify=g_df.Target,
    test_size=0.2,
    random_state=124,
)
```

### 모델 학습

```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

m_nb_pipe = Pipeline([
    ("count_vectorizer", CountVectorizer()),
    ("multinomial_NB", MultinomialNB()),
])

m_nb_pipe.fit(X_train.values, y_train)
prediction = m_nb_pipe.predict(X_test.values)
prediction_proba = m_nb_pipe.predict_proba(X_test.values)
```

### 모델 비교

| 모델 | Accuracy | Precision | Recall | F1 | AUC |
|---|---:|---:|---:|---:|---:|
| **CountVectorizer + MultinomialNB** | **0.9406** | **0.9429** | **0.9413** | **0.9408** | **0.9946** |
| CountVectorizer + DecisionTree | 0.9147 | 0.9176 | 0.9165 | 0.9150 | 0.9530 |

![Naive Bayes 분류 결과](images/globalgates_category_classifier_01.png)

![Decision Tree 비교 결과](images/globalgates_category_classifier_02.png)

Naive Bayes를 선택한 이유는 단순히 정확도가 높아서가 아닙니다.
상품 등록 화면에서는 정답 하나보다 후보 3개를 보여주는 UX가 더 자연스럽기 때문에, 카테고리별 확률 ranking이 안정적인 모델이 필요했습니다.
AUC가 0.9946으로 높았기 때문에 Top-3 추천 방식에 적합하다고 판단했습니다.

### 모델 저장

```python
import joblib

joblib.dump(m_nb_pipe, "globalgates_category_model.pkl")
joblib.dump(category_encoder, "globalgates_category_encoder.pkl")
```

### FastAPI 적용 코드

```python
text = " ".join(value for value in [post_title, post_content, post_tag] if value)
probabilities = model.predict_proba([text])[0]
class_ids = list(model.classes_)

ranked_indices = sorted(
    range(len(probabilities)),
    key=lambda i: probabilities[i],
    reverse=True,
)[:3]
```

```python
predictions.append(
    CategoryPredictionItem(
        categoryName=category_name,
        score=round(float(probabilities[i]), 4),
    )
)
```

### API

| 항목 | 내용 |
|---|---|
| FastAPI endpoint | `POST /api/ai/category/predict` |
| 입력 | `postTitle`, `postContent`, `postTag` |
| 출력 | `categoryName`, `score`가 포함된 Top-3 목록 |

---

## 4. 팔로우 추천

> **목표**: 회원의 자기소개와 관심 카테고리를 바탕으로 연결 가능성이 높은 회원을 추천합니다.

### 추천 데이터 조회

추천은 서비스 DB에 있는 회원, 팔로우, 관심 카테고리 데이터를 사용했습니다.

```python
follow_df = pd.read_sql(
    "SELECT follower_id, following_id FROM tbl_follow",
    engine,
)

category_map_df = pd.read_sql(
    """
    SELECT mcr.member_id, c.category_name
    FROM tbl_member_category_rel mcr
    JOIN tbl_category c ON c.id = mcr.category_id
    """,
    engine,
)
```

| 데이터 | 규모 | 사용 목적 |
|---|---:|---|
| 회원 데이터 | 508명 | 추천 후보 |
| 팔로우 관계 | 2,422건 | 이미 팔로우한 회원 제외 |
| 회원-카테고리 관계 | 1,605건 | 관심사 유사도 계산 |

### 프로필 텍스트 구성

회원의 `bio`만 사용하면 정보가 부족한 경우가 많기 때문에, 관심 카테고리까지 합쳐서 추천용 텍스트를 만들었습니다.

```python
from konlpy.tag import Okt

okt = Okt()

category_text_df = (
    category_map_df.groupby("member_id")["category_name"]
    .apply(lambda x: " ".join(sorted(set(x))))
    .reset_index(name="category_text")
)

df = df.merge(category_text_df, left_on="id", right_on="member_id", how="left")
df["member_bio"] = df["member_bio"].fillna("")
df["category_text"] = df["category_text"].fillna("")

df["bio_text"] = df["member_bio"].apply(lambda x: " ".join(okt.nouns(x)) if x else "")
df["intro_text"] = (df["bio_text"] + " " + df["category_text"]).str.strip()
```

### 유사도 계산

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

tfidf_v = TfidfVectorizer()
tfidf_matrix = tfidf_v.fit_transform(pre_df["intro"])
cosine_full = cosine_similarity(tfidf_matrix)

print(cosine_full.shape)
```

노트북 검증에서는 `508 × 508` 유사도 행렬을 만들고, 기준 회원과 비슷한 회원을 확인했습니다.

![팔로우 추천 검증](images/globalgates_follower_recommender_01.png)

![팔로우/인기도 분포](images/globalgates_follower_recommender_profile_score_01.png)

### 후보 제외 로직

추천 결과에는 자기 자신과 이미 팔로우한 회원이 나오면 안 됩니다.
운영 코드에서는 여기에 차단 관계와 비활성 회원까지 제외했습니다.

```python
member_id = int(df["id"].iloc[1])
self_idx = np.where(df["id"].values == member_id)[0][0]

sim_scores = cosine_full[self_idx]
sim_indices = sim_scores.argsort()[::-1]

followed_ids = follow_df[follow_df.follower_id == member_id].following_id.values
exclude_ids = np.append(followed_ids, member_id)

mask = ~np.isin(df.iloc[sim_indices]["id"].values, exclude_ids)
top5_indices = sim_indices[mask][:5]
```

FastAPI 운영 코드에서는 DB 조회 단계에서 이미 팔로우/차단/비활성 회원을 제외합니다.

```sql
where m.member_status = 'active'
  and m.id != $1
  and m.id not in (
      select f.following_id
      from tbl_follow f
      where f.follower_id = $1
  )
  and m.id not in (
      select b.blocked_id
      from tbl_block b
      where b.blocker_id = $1
  )
  and m.id not in (
      select b.blocker_id
      from tbl_block b
      where b.blocked_id = $1
  )
```

### FastAPI 추천 코드

```python
me_text = self.build_text(me)

if me_text:
    rows = self.rank_with_tfidf(me_text, rows)
else:
    rows.sort(key=lambda row: int(row.get("follower_count", 0)), reverse=True)
    for row in rows:
        row["score"] = float(row.get("follower_count", 0))
        row["candidate_source"] = "cold_start"
```

```python
common = set(me_counter) & set(counter)
dot = sum(me_counter[word] * counter[word] for word in common)
return dot / (me_norm * norm)
```

---

## 5. 무역 문서 RAG 챗봇

> **목표**: 사용자가 무역 PDF를 직접 읽지 않아도 질문으로 필요한 정보를 찾을 수 있게 합니다.

### 문서 로드

무역 문서는 국가와 기관 정보를 metadata로 붙여서 불러왔습니다.

```python
all_docs = []

for file_path in FILE_PATHS:
    loader = PyMuPDFLoader(file_path)
    docs = loader.load()

    pdf_path = Path(file_path)
    relative_parts = pdf_path.relative_to(DOCUMENT_ROOT).parts
    country_code = relative_parts[0] if len(relative_parts) >= 3 else ""
    agency_code = relative_parts[1] if len(relative_parts) >= 3 else ""

    for doc in docs:
        doc.metadata.update({
            "country_code": country_code,
            "agency_code": agency_code,
            "file_name": pdf_path.name,
            "local_path": file_path,
        })

    all_docs.extend(docs)
```

| 항목 | 값 |
|---|---:|
| 실습 기준 PDF | 10개 |
| 전체 페이지 | 315페이지 |
| 청크 수 | 1,469개 |
| 청크 크기 | 500자 |
| 청크 overlap | 50자 |
| 임베딩 모델 | `jhgan/ko-sbert-nli` |

### 문서 분할과 벡터 DB 생성

```python
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
)

split_documents = text_splitter.split_documents(all_docs)
print(f"분할된 청크(조각)의 수: {len(split_documents)}")
```

```python
embeddings = HuggingFaceEmbeddings(
    model_name="jhgan/ko-sbert-nli",
    model_kwargs={"device": "cpu"},
)

vectorstore = Redis.from_documents(
    documents=split_documents,
    embedding=embeddings,
    redis_url="redis://localhost:6380",
    index_name="trade_regulation_docs",
)
```

### 검색과 답변 생성

```python
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 8, "fetch_k": 20},
)

llm = ChatOpenAI(
    model_name="gpt-5.4-nano",
    temperature=0,
)
```

프롬프트에서는 문서 밖의 내용을 추측하지 않도록 제한했습니다.

```python
prompt_template = PromptTemplate.from_template(
    """
    너는 국가별 수출입 법규와 통관 실무를 정리해주는 무역 브리핑 비서다.

    아래 문맥(Context)에 포함된 정보를 바탕으로 답변하라.
    답변은 문맥에 나온 정보만 사용해야 하며, 문맥 밖의 사실을 추측해서 추가하지 마라.

    #Context:
    {context}

    #Question:
    {question}

    #Answer:
    """
)
```

### FastAPI 문서 적재

서비스에서는 관리자가 PDF를 업로드하면 Spring Boot가 S3에 저장하고, FastAPI가 S3 key를 받아 RAG 인덱싱을 수행합니다.

```python
@router.post("/ingest", response_model=RagIngestResponse)
async def ingest(request: RagIngestRequest):
    await rag_service.ingest_document_from_s3(request.s3Key)
    return RagIngestResponse(message="문서 적재가 완료되었습니다.")
```

```python
with tempfile.TemporaryDirectory(prefix="globalgates-rag-") as temp_dir:
    local_path = Path(temp_dir) / f"source{suffix}"
    download_from_s3(cleaned, local_path)
    await self.ingest_document(str(local_path))
```

### Redis semantic cache

같거나 비슷한 질문은 Redis cache에서 먼저 찾습니다.
캐시가 있으면 LLM 호출 없이 바로 응답하고, 없으면 RAG 답변을 생성한 뒤 저장합니다.

```python
cached_answer, _score = search_similar_question(cleaned)
if cached_answer is not None:
    return {
        "answer": cached_answer,
        "cached": True,
        "sources": [],
    }

answer = await rag_service.ask(cleaned)
save_question_answer(cleaned, answer)
```

---

## 6. n8n 뉴스 자동화

> **목표**: 외부 경제 뉴스를 자동 수집하고, OpenAI로 요약해 서비스 운영 데이터로 저장합니다.

뉴스 수집과 요약은 사람이 반복해서 처리할 필요가 없는 업무입니다.
n8n에서는 RSS 수집, 기사 본문 요청, HTML 추출, OpenAI 요약, PostgreSQL 저장을 하나의 워크플로우로 구성했습니다.

| 노드 | 설정 |
|---|---|
| Schedule Trigger | 매일 09:00 실행 |
| Webhook | 수동 실행 또는 외부 호출 |
| RSS Read | `https://www.mk.co.kr/rss/30100041/` |
| HTTP Request | 기사 링크 본문 요청 |
| HTML | `.view_head_title`, `.news_cnt_detail_wrap` 추출 |
| OpenAI | `gpt-5.4-nano`로 요약 |
| PostgreSQL | `tbl_news.news_contents` 저장 |

### n8n Code Node

```javascript
const allItems = $input.all();

let combinedText = allItems.map((item, index) => {
  const title = item.json.title || '제목 없음';
  const content = item.json.contentSnippet || item.json.content || '';
  return `[뉴스 ${index + 1}] 제목: ${title}\n내용: ${content}`;
}).join('\n\n---\n\n');

combinedText = combinedText.replaceAll("#", "");
combinedText = combinedText.replaceAll("*", "");

return [{
  json: {
    totalNewsContent: combinedText,
    count: allItems.length
  }
}];
```

### 요약 프롬프트

```text
너는 경제 뉴스 요약기이다. 아래 기사를 한국어로 간결하게 요약하라.

[요약 규칙]
1) 뉴스별로 1줄 요약
2) '1) [뉴스 1]'과 같은 구분점 없이 오로지 줄바꿈으로만 구분한다.
3) 과장/추측 금지, 기사에 있는 사실만
4) 투자 추천/매수·매도 조언 금지
```

---

## 7. Spring Boot와 FastAPI 연동

### 역할 분리

| 서버/도구 | 역할 |
|---|---|
| Spring Boot | 화면, 로그인, 상품/회원 비즈니스 로직 |
| FastAPI | AI 모델 추론, 추천, RAG 질의 |
| PostgreSQL | 회원, 상품, 팔로우, 뉴스 저장 |
| AWS S3 | RAG 문서 파일 저장 |
| Redis | 챗봇 semantic cache |
| n8n | 뉴스 수집·요약 자동화 |

### 주요 API

| 기능 | FastAPI API | 결과 |
|---|---|---|
| 상품 카테고리 추천 | `POST /api/ai/category/predict` | 카테고리 Top-3 |
| 팔로우 추천 | `POST /api/ai/follow/recommend` | 추천 회원 목록 |
| RAG 문서 적재 | `POST /api/rag/ingest` | 문서 인덱싱 |
| 챗봇 질문 | `POST /api/chat/query` | 답변 + cache 여부 |
| RAG 직접 질의 | `POST /api/rag/query` | RAG 답변 |

### FastAPI 초기화

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await db.connect()
    ai_service.load_follow_artifacts()
    await rag_service.initialize()
    yield
    await db.disconnect()
```

---

## 8. 최종 정리

| 파트 | 핵심 기술 | 정량 근거 | 서비스 연결 |
|---|---|---:|---|
| 기획 분석 | pandas, matplotlib | 중소기업 99.87%, 대기업 수출액 64.49% | 서비스 문제 정의 |
| 상품 분류 | CountVectorizer + MultinomialNB | Accuracy 0.9406, AUC 0.9946 | 상품 등록 카테고리 추천 |
| 팔로우 추천 | TF-IDF, cosine similarity | 회원 508명, 팔로우 2,422건 | 추천 회원 카드 |
| RAG 챗봇 | Redis, HuggingFace Embedding, LLM | 315페이지, 1,469청크 | 무역 문서 질의 |
| n8n 자동화 | RSS, OpenAI, PostgreSQL | RSS → 요약 → DB 저장 | 경제 뉴스 요약 |
| 연동 | Spring Boot + FastAPI | 주요 AI API 5개 | 실제 서비스 화면 연결 |

### 발표에서 강조할 문장

1. **기획 배경은 감이 아니라 KOSIS/관세청 데이터를 분석해서 잡았습니다.**
2. **중소기업은 수가 적은 것이 아니라, 수출 성과와 연결 기회가 대기업에 비해 약하게 분산되어 있습니다.**
3. **상품 분류는 17만 건 이상의 텍스트로 학습했고, Top-3 추천에 맞는 확률 기반 모델을 선택했습니다.**
4. **팔로우 추천은 프로필, 관심 카테고리, 팔로우/차단 관계를 함께 반영했습니다.**
5. **RAG와 n8n은 사용자가 직접 문서와 뉴스를 찾아보는 시간을 줄이기 위한 기능입니다.**

### 한 줄 결론

> GlobalGates AI는 데이터 분석으로 확인한 중소기업 수출 구조의 문제를 바탕으로, 상품 노출·회원 연결·무역 정보 탐색·뉴스 확인을 실제 서비스 기능으로 자동화한 AI 파트입니다.
