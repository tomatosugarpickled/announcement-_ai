# GlobalGates AI 발표 자료

> **피드로 여는 기업용 비즈니스 소셜 마켓**
>
> GlobalGates는 국내 중소기업과 해외 바이어를 연결하는 B2B 소셜 마켓입니다.
>
> AI 파트는 상품 등록, 회원 연결, 무역 문서 탐색, 경제 뉴스 확인처럼 서비스 안에서 반복되는 판단과 검색 과정을 줄이는 데 초점을 맞췄습니다.

---

## 목차

1. [기획 배경](#1-기획-배경)
2. [AI 적용 범위](#2-ai-적용-범위)
3. [데이터 준비](#3-데이터-준비)
4. [상품 카테고리 분류](#4-상품-카테고리-분류)
5. [팔로우 추천](#5-팔로우-추천)
6. [무역 문서 RAG 챗봇](#6-무역-문서-rag-챗봇)
7. [n8n 뉴스 자동화](#7-n8n-뉴스-자동화)
8. [Spring Boot와 FastAPI 연동](#8-spring-boot와-fastapi-연동)
9. [최종 정리](#9-최종-정리)

---

## 1. 기획 배경

### 왜 이 서비스에 AI가 필요한가

GlobalGates는 중소기업이 해외 시장에 진출할 때 필요한 사람, 상품, 정보가 한곳에서 연결되는 서비스를 목표로 합니다.

하지만 실제 수출 과정에서는 단순히 게시글을 올리고 검색창을 제공하는 것만으로는 부족합니다. 상품을 등록하는 사람은 적절한 카테고리를 고르는 데 시간을 쓰고, 바이어나 셀러를 찾는 사람은 수많은 회원 중 누구와 연결해야 할지 판단해야 합니다. 무역 문서를 확인해야 하는 상황에서는 PDF를 직접 읽어야 하고, 시장 상황을 따라가려면 뉴스를 계속 확인해야 합니다.

그래서 AI를 별도의 부가 기능으로 붙인 것이 아니라, 사용자가 서비스 안에서 막히는 지점에 배치했습니다.

| 사용자가 막히는 지점 | 서비스에서 필요한 도움 | 적용한 AI 기능 |
|---|---|---|
| 상품을 어느 카테고리에 올릴지 애매함 | 등록 과정에서 후보를 먼저 제시 | 상품 카테고리 Top-3 추천 |
| 누구와 연결해야 할지 모름 | 관심사와 프로필이 가까운 회원 추천 | 팔로우 추천 |
| 무역 문서를 직접 읽기 어려움 | 문서 안에서 질문과 관련된 내용 검색 | RAG 챗봇 |
| 경제 뉴스를 계속 확인하기 어려움 | 주요 뉴스를 자동 수집하고 요약 | n8n 뉴스 요약 |

![중소기업 수출 구조 분석](images/export_gap_analysis_01.png)

![중소기업 수출 추이](images/export_gap_analysis_02.png)

### 발표에서 가져갈 관점

이번 AI 파트의 핵심은 “모델을 만들었다”가 아닙니다.

중요한 점은 모델과 자동화가 실제 서비스 화면과 API에 연결되어 있다는 것입니다. 상품 등록 화면에서는 카테고리를 추천하고, 회원 영역에서는 팔로우 후보를 추천하고, 챗봇에서는 무역 문서 기반 답변을 제공하고, n8n은 외부 뉴스를 요약해 운영 데이터로 저장합니다.

즉, GlobalGates AI는 사용자가 다음 행동을 더 쉽게 결정하도록 돕는 기능입니다.

---

## 2. AI 적용 범위

| 구분 | 사용자 입력 | AI 처리 | 서비스 결과 |
|---|---|---|---|
| 상품 분류 | 상품명, 본문, 태그 | 카테고리별 확률 계산 | 추천 카테고리 Top-3 |
| 팔로우 추천 | 회원 bio, 관심 카테고리, 팔로우/차단 관계 | 후보 제외 후 유사도 계산 | 추천 회원 목록 |
| RAG 챗봇 | 무역 PDF, 사용자 질문 | 문서 검색 후 답변 생성 | 문서 기반 한국어 답변 |
| 뉴스 자동화 | RSS 기사 | 본문 추출, OpenAI 요약 | 뉴스 요약 DB 저장 |

| 서버/도구 | 담당 역할 |
|---|---|
| Spring Boot | 화면, 로그인, 상품/회원 비즈니스 로직 |
| FastAPI | 상품 분류, 팔로우 추천, RAG 질의 |
| PostgreSQL | 회원, 상품, 팔로우, 뉴스 데이터 저장 |
| AWS S3 | RAG 문서 파일 저장 |
| Redis | 챗봇 semantic cache |
| n8n | RSS 뉴스 수집과 요약 자동화 |

---

## 3. 데이터 준비

AI 기능별로 필요한 데이터가 달랐기 때문에 외부 데이터, 서비스 DB 데이터, 문서 데이터를 나눠서 준비했습니다.

| 데이터 | 규모 | 사용 위치 | 의미 |
|---|---:|---|---|
| 네이버 쇼핑 OpenAPI | 70,000건 | 상품 분류 | 상품명·카테고리 중심 텍스트 |
| 네이버 뉴스/블로그 OpenAPI | 112,000건 | 상품 분류 보강 | 수출·수입·물류·관세·금융 텍스트 |
| 최종 분류 학습 데이터 | 176,426건 | Naive Bayes 학습 | 상품/무역 텍스트 통합 학습셋 |
| CountVectorizer 어휘 | 396,276개 | 분류 모델 | 문장을 숫자 벡터로 변환 |
| 회원 데이터 | 508명 | 팔로우 추천 | 추천 대상 회원 풀 |
| 팔로우 관계 | 2,422건 | 팔로우 추천 | 이미 연결된 회원 제외 |
| 회원-카테고리 관계 | 1,605건 | 팔로우 추천 | 관심사 기반 유사도 |
| 무역 PDF | 315페이지 | RAG | 문서 기반 답변 소스 |
| RAG 청크 | 1,469개 | RAG | 검색 단위 |

### 전처리 방식

| 원본 | 전처리 | 이유 |
|---|---|---|
| 상품명 + 설명 + 태그 | 하나의 문장으로 결합 | 상품의 전체 맥락을 모델 입력으로 사용 |
| 회원 bio + 관심 카테고리 | 프로필 문장으로 결합 | 비슷한 목적의 회원을 찾기 위해 사용 |
| PDF 문서 | 500자 청크, 50자 overlap | 검색할 때 문맥이 끊기지 않도록 구성 |
| RSS 기사 | 제목·본문 추출 후 특수문자 정리 | 요약 프롬프트 입력 안정화 |

---

## 4. 상품 카테고리 분류

> **목표**: 상품 등록 시 사용자가 직접 카테고리를 찾지 않아도, 상품명·설명·태그만으로 적합한 카테고리 후보를 제시합니다.

### 모델 선택 이유

상품 카테고리는 사용자가 최종 선택할 수 있어야 하기 때문에, 모델이 정답 하나만 반환하는 방식보다 후보를 여러 개 보여주는 방식이 더 적합했습니다. 그래서 카테고리별 확률을 계산하고, 확률이 높은 3개를 화면에 보여주는 구조로 만들었습니다.

| 모델 | Accuracy | Precision | Recall | F1 | AUC |
|---|---:|---:|---:|---:|---:|
| **CountVectorizer + MultinomialNB** | **0.9406** | **0.9429** | **0.9413** | **0.9408** | **0.9946** |
| CountVectorizer + DecisionTree | 0.9147 | 0.9176 | 0.9165 | 0.9150 | 0.9530 |

Naive Bayes를 선택한 이유는 다음과 같습니다.

| 기준 | 판단 |
|---|---|
| 정확도 | Decision Tree보다 높음 |
| F1 | 전체 카테고리 기준으로 더 안정적 |
| AUC | 0.9946으로 Top-3 확률 추천에 적합 |
| 과적합 | train/test 차이가 2.7%p 수준 |

![Naive Bayes 분류 결과](images/globalgates_category_classifier_01.png)

![Decision Tree 비교 결과](images/globalgates_category_classifier_02.png)

### 학습 코드 핵심

```python
m_nb_pipe = Pipeline([
    ("count_vectorizer", CountVectorizer()),
    ("multinomial_NB", MultinomialNB()),
])

m_nb_pipe.fit(X_train.values, y_train)
prediction = m_nb_pipe.predict(X_test.values)
proba = m_nb_pipe.predict_proba(X_test.values)
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
CategoryPredictionItem(
    categoryName=category_name,
    score=round(float(probabilities[i]), 4),
)
```

### 서비스 적용 결과

| 단계 | 내용 |
|---|---|
| 사용자 입력 | 상품명, 상품 설명, 태그 |
| FastAPI 처리 | 세 텍스트를 합쳐 모델 입력 생성 |
| 모델 결과 | 카테고리별 확률 계산 |
| 화면 표시 | 확률이 높은 카테고리 3개를 추천 칩으로 표시 |

---

## 5. 팔로우 추천

> **목표**: 사용자가 GlobalGates 안에서 연결할 만한 바이어·셀러 후보를 빠르게 찾도록 돕습니다.

### 추천 기능이 필요한 이유

GlobalGates는 상품만 등록하는 마켓이 아니라 회원 간 연결이 중요한 비즈니스 소셜 서비스입니다. 사용자가 팔로우할 대상을 잘 찾으면 피드에서 보는 정보와 거래 가능성이 달라집니다. 그래서 단순 인기순이 아니라, 회원의 자기소개와 관심 카테고리를 반영한 추천이 필요했습니다.

### 사용 데이터

| 데이터 | 규모 | 역할 |
|---|---:|---|
| 회원 데이터 | 508명 | 추천 후보 |
| 팔로우 관계 | 2,422건 | 이미 팔로우한 회원 제거 |
| 회원-카테고리 관계 | 1,605건 | 관심 분야 비교 |
| 유사도 행렬 | 508 × 508 | 노트북 검증 |

![팔로우 추천 검증](images/globalgates_follower_recommender_01.png)

![팔로우/인기도 분포](images/globalgates_follower_recommender_profile_score_01.png)

### 추천 방식

노트북에서는 `bio + category` 텍스트에 TF-IDF와 코사인 유사도를 적용해 추천 가능성을 확인했습니다. 운영 코드에서는 현재 회원 수와 응답 속도를 고려해, FastAPI에서 후보를 조회한 뒤 토큰 빈도 기반 코사인 유사도로 점수를 계산합니다.

추천 후보를 만들 때는 먼저 서비스 정책상 보여주면 안 되는 회원을 제외합니다.

| 제외 조건 | 이유 |
|---|---|
| 자기 자신 | 추천 대상이 될 수 없음 |
| 이미 팔로우한 회원 | 중복 추천 방지 |
| 내가 차단한 회원 | 사용자 의사 반영 |
| 나를 차단한 회원 | 상대방 의사 반영 |
| 비활성 회원 | 실제 연결 불가능 |

### 운영 코드 핵심

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

### 응답 예시

```json
{
  "recommendations": [
    {
      "memberId": 2,
      "categoryText": "수출 물류",
      "score": 0.8462,
      "rankPosition": 1,
      "candidateSource": "tfidf"
    }
  ]
}
```

---

## 6. 무역 문서 RAG 챗봇

> **목표**: 사용자가 무역 PDF를 직접 읽지 않아도 질문으로 필요한 정보를 찾을 수 있게 합니다.

### RAG를 사용한 이유

무역 문서는 정책, 절차, 서류명, 조건, 기관명처럼 정확성이 중요한 정보가 많습니다. LLM이 일반 지식으로 답하면 틀릴 수 있기 때문에, 먼저 문서에서 관련 내용을 찾고 그 근거 안에서만 답하도록 RAG 구조를 사용했습니다.

### 처리 규모

| 항목 | 값 |
|---|---:|
| 실습 기준 PDF | 10개 |
| 전체 페이지 | 315페이지 |
| 청크 수 | 1,469개 |
| 청크 크기 | 500자 |
| 청크 overlap | 50자 |
| 임베딩 차원 | 768 |
| 검색 방식 | Hybrid Query |

### 문서 적재

관리자가 PDF를 업로드하면 Spring Boot가 파일을 S3에 저장합니다. 이후 FastAPI가 S3 key를 받아 문서를 내려받고, RAG 엔진이 문서를 파싱해 검색 가능한 형태로 저장합니다.

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

### 질문 처리

질문이 들어오면 먼저 Redis semantic cache에서 비슷한 질문이 있는지 확인합니다. 캐시에 없으면 RAG 검색을 수행하고, 답변을 생성한 뒤 다음 요청을 위해 Redis에 저장합니다.

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

```python
result = await self._rag.aquery(
    cleaned,
    mode="hybrid",
    system_prompt=load_rag_system_prompt(),
)
```

### 프롬프트 정책

| 원칙 | 내용 |
|---|---|
| 문서 근거 제한 | 제공된 Context 안의 정보만 사용 |
| 추측 금지 | 문서에 없으면 만들지 않음 |
| 확인 불가 응답 | "제공된 문서에서는 확인할 수 없습니다."라고 답변 |
| 한국어 응답 | 모든 답변은 한국어 |
| 숫자·날짜 보존 | 문서의 수치, 날짜, 기관명을 최대한 그대로 유지 |

---

## 7. n8n 뉴스 자동화

> **목표**: 외부 경제 뉴스를 자동 수집하고, OpenAI로 요약해 서비스 운영 데이터로 저장합니다.

### n8n을 사용한 이유

뉴스 수집과 요약은 반복 업무입니다. 매번 사람이 기사를 열어보고 정리하는 대신, n8n으로 RSS 수집, 본문 추출, 요약, DB 저장을 자동화했습니다.

### 실제 구성

| 노드 | 설정 |
|---|---|
| Schedule Trigger | 매일 09:00 실행 |
| Webhook | 수동 실행 또는 외부 호출 |
| RSS Read | `https://www.mk.co.kr/rss/30100041/` |
| HTTP Request | 기사 링크 본문 요청 |
| HTML | `.view_head_title`, `.news_cnt_detail_wrap` 추출 |
| OpenAI | `gpt-5.4-nano`로 요약 |
| PostgreSQL | `tbl_news.news_contents` 저장 |

### 요약 프롬프트

```text
너는 경제 뉴스 요약기이다. 아래 기사를 한국어로 간결하게 요약하라.

[요약 규칙]
1) 뉴스별로 1줄 요약
2) '1) [뉴스 1]'과 같은 구분점 없이 오로지 줄바꿈으로만 구분한다.
3) 과장/추측 금지, 기사에 있는 사실만
4) 투자 추천/매수·매도 조언 금지
```

이 기능은 모델 학습 기능은 아니지만, 서비스 외부의 경제 정보를 자동으로 수집하고 요약해 운영 데이터로 바꾸는 자동화 사례입니다.

---

## 8. Spring Boot와 FastAPI 연동

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

## 9. 최종 정리

### AI 파트 요약

| 파트 | 핵심 기술 | 정량 근거 | 서비스 연결 |
|---|---|---:|---|
| 상품 분류 | CountVectorizer + MultinomialNB | Accuracy 0.9406 / AUC 0.9946 | 상품 등록 카테고리 추천 |
| 팔로우 추천 | 프로필·관심사 코사인 유사도 | 회원 508명 / 관계 2,422건 | 추천 회원 카드 |
| RAG 챗봇 | RAGAnything + LightRAG + Redis | 315페이지 / 1,469청크 | 무역 문서 질의 |
| n8n 자동화 | RSS + OpenAI + PostgreSQL | RSS → 요약 → DB 저장 | 경제 뉴스 요약 |
| 연동 | Spring Boot + FastAPI | 주요 AI API 5개 | 실제 서비스 화면 연결 |

### 발표에서 강조할 문장

1. **AI를 별도 데모가 아니라 서비스 안의 실제 사용 지점에 연결했습니다.**
2. **상품 분류는 17만 건 이상의 학습 데이터로 Top-3 카테고리 추천을 구현했습니다.**
3. **팔로우 추천은 단순 인기순이 아니라 프로필, 관심사, 팔로우/차단 관계를 함께 반영했습니다.**
4. **RAG 챗봇은 문서 근거 안에서만 답하도록 제한해 환각 위험을 줄였습니다.**
5. **n8n은 외부 경제 뉴스를 자동으로 운영 데이터로 전환한 자동화 사례입니다.**

### 한 줄 결론

> GlobalGates AI는 상품을 분류하고, 사람을 연결하고, 문서를 검색하고, 뉴스를 요약해 중소기업의 글로벌 B2B 활동을 더 빠르게 만드는 서비스형 AI 구조입니다.
