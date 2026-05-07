---
title: '벡터 DB 2편: 청킹과 임베딩 전략 — RAG 품질을 결정하는 것들'
description: '같은 벡터 DB를 써도 검색 정확도가 60%와 92%로 갈리는 이유가 있어요. DB 선택보다 더 중요한 것 — 데이터를 어떻게 준비하느냐를 다뤄요.'
pubDate: '2026-05-06'
heroImage: '../../assets/vector-db-chunking_hero_image.png'
tags: ['Vector DB', 'RAG', 'Chunking', 'Embedding', 'AI-Ready']
---

같은 Pinecone을 쓰는 두 팀이 있었어요.

팀 A는 문서를 통째로 임베딩했어요. 검색 정확도는 60%였고, 답변이 느리고 맥락이 오염됐거든요. 팀 B는 청킹 전략을 설계하고, 메타데이터를 꼼꼼히 붙이고, 임베딩 모델을 신중하게 골랐어요. 결과는 92%였어요.

벡터 DB는 그릇이에요. 무엇을 어떻게 담느냐가 음식의 맛을 결정하죠. 이번 편은 그 차이를 만드는 것들 — 청킹, 메타데이터, 임베딩 모델 선택을 다뤄요.

## 왜 전처리가 RAG 품질을 결정하나요

많은 팀이 벡터 DB 선택에 시간을 쏟지만, 실제 품질은 **무엇을 어떻게 넣느냐**에서 결정돼요.

```
[같은 Pinecone을 쓰는 두 팀]

팀 A: 문서 통째로 임베딩, 메타데이터 없음
  → 검색 정확도 60%, 느린 답변, 맥락 오염

팀 B: 청킹 전략 설계, 메타데이터 풍부, 모델 선택 신중
  → 검색 정확도 92%, 정확한 답변, 빠른 응답
```

직접 겪어보니 더라고요. 처음에 저도 "일단 넣고 보자" 방식으로 시작했다가, 관련 없는 문서가 계속 올라오는 걸 보고 전처리를 다시 설계했어요. 개선 전후가 하늘과 땅 차이였거든요.

그럼 구체적으로 어떻게 해야 하는지 볼게요.

---

## 청킹 전략 — 문서를 어떻게 자르는가

### 왜 청킹이 필요한가요

임베딩 모델에는 입력 길이 제한이 있어요. 그리고 긴 문서 전체를 하나의 벡터로 만들면 의미가 희석되거든요.

```
[문서 전체를 하나의 벡터로]
"QA 프로세스 문서 (50페이지)" → 벡터 1개
→ "flaky test 대응 방법" 질문 → 50페이지 중 관련 내용 1단락인데
  전체 문서 벡터가 반환됨 → LLM이 50페이지를 받음 → 맥락 오염, 비용 폭증

[청킹 후 임베딩]
"QA 프로세스 문서" → 청크 200개 → 벡터 200개
→ 관련 청크 5개만 반환 → LLM이 딱 필요한 내용만 받음
```

청킹 방식은 크게 네 가지예요.

### Fixed Size Chunking — 가장 단순하게

글자 수나 토큰 수 기준으로 일정하게 자르는 방식이에요.

```python
def fixed_chunk(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap  # overlap으로 문맥 연결
    return chunks
```

구현이 단순하고 청크 수가 예측 가능해요. 단점은 문장 중간에서 잘릴 수 있다는 거예요. 빠른 프로토타입이나 균일한 구조의 문서에 적합해요.

### Semantic Chunking — 의미 단위로

문장 의미가 바뀌는 지점을 감지해서 자르는 방식이에요.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai.embeddings import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95
)
chunks = splitter.split_text(document)
```

의미 단절이 없고 자연스러운 청크가 만들어져요. 다만 임베딩 비용이 추가로 들고 처리 시간도 길어지거든요. 긴 서술형 문서나 높은 품질이 필요한 경우에 써요.

### Hierarchical Chunking — 문서 구조를 살려서

헤더 구조를 유지하며 청크를 만드는 방식이에요. Confluence, Notion, GitHub README처럼 구조화된 문서에 딱 맞아요.

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "H1"), ("##", "H2"), ("###", "H3")
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on)
chunks = splitter.split_text(markdown_document)
# 각 청크에 헤더 경로가 메타데이터로 포함됨
```

문서 구조가 보존되고, 헤더 경로가 메타데이터로 자동 추가돼요. 사내 문서 RAG에서는 이 방식이 가장 효과적이었어요.

### Sliding Window — 경계 맥락을 보강

고정 크기에 overlap을 크게 줘서 문맥 손실을 최소화해요.

```
청크 크기: 400 토큰
Overlap: 150 토큰

청크 1: 토큰 0~400
청크 2: 토큰 250~650   ← 150 토큰 중복
청크 3: 토큰 500~900
```

청크 경계에서 맥락이 끊기는 걸 방지하지만, 중복으로 저장 용량이 늘어요. 연속적 맥락이 중요한 문서 — 법률, 계약서, 긴 설명서에 적합해요.

### 청크 크기 설계 기준

| 항목 | 너무 작으면 | 너무 크면 | 권장 범위 |
| --- | --- | --- | --- |
| 청크 크기 | 맥락 부족, 단어 단편화 | 의미 희석, LLM 비용 증가 | 256~512 토큰 |
| Overlap | 경계에서 맥락 단절 | 중복 증가, 검색 노이즈 | 청크의 10~20% |
| Top-K 반환 수 | 정보 부족 | LLM 컨텍스트 초과, 노이즈 | 3~7개 |

실용적인 시작점은 이렇게 잡으면 돼요:

```
일반 기술 문서: 청크 400토큰, Overlap 80토큰
FAQ / 짧은 항목: 청크 200토큰, Overlap 없음
긴 설명서 / 계약서: 청크 600토큰, Overlap 120토큰
```

---

## 특수 콘텐츠 처리

코드나 표처럼 단순 텍스트 변환으로는 의미가 손실되는 콘텐츠들이 있어요.

### 표(Table) 처리

표를 그냥 텍스트로 변환하면 관계가 무너져버려요.

```
❌ 나쁜 방법 (단순 텍스트 변환)
"항목 Pinecone Supabase 벡터검색 최고 좋음 일반DB 없음 있음"
→ 임베딩이 관계를 파악하지 못함

✅ 좋은 방법 (행 단위로 자연어 변환)
"Pinecone은 벡터 검색이 최고 수준이지만 일반 DB 기능은 없다."
"Supabase는 벡터 검색 품질이 좋고 일반 PostgreSQL DB 기능도 지원한다."
→ 각 행이 의미 있는 문장이 됨
```

### 코드 블럭 처리

코드는 별도 청크로 분리하고, 언어와 목적을 메타데이터로 붙여요.

```python
{
    "id": "doc-001-code-1",
    "content": "def embed(text): ...",
    "metadata": {
        "type": "code",
        "language": "python",
        "context": "Pinecone 임베딩 함수",
        "parent_section": "기본 사용법"
    }
}
```

---

## 메타데이터 설계

벡터 검색만으로는 특정 범위를 제한하기 어려워요. 메타데이터 필터를 사용하면 검색 공간을 좁혀서 정확도와 속도를 동시에 높일 수 있어요.

```python
results = index.query(
    vector=embed("E2E 테스트 방법"),
    filter={
        "team": "QA",
        "doc_type": "guide",
        "updated_after": "2026-01-01"
    },
    top_k=5
)
# QA팀 가이드 문서 중 2026년 이후 업데이트된 것만 검색
```

필드 설계는 아래를 기준으로 시작하면 돼요:

| 필드 | 예시 값 | 용도 |
| --- | --- | --- |
| `source` | confluence, github, notion | 출처 필터 |
| `doc_type` | guide, faq, report, spec | 문서 유형 필터 |
| `team` | QA, FE, BE, PM | 팀별 검색 |
| `created_at` | 2026-04-27 | 날짜 범위 필터 |
| `language` | ko, en | 언어 필터 |
| `parent_title` | "벡터 DB 전처리 가이드" | 원본 문서 제목 |
| `chunk_index` | 3 | 문서 내 위치 |

설계 원칙은 간단해요. **자주 필터링할 것만 메타데이터로** — 너무 많으면 오버헤드만 늘어요. 기수(cardinality)가 낮은 값(팀명, 문서 유형)을 선호하고, 날짜는 ISO 8601로 통일하는 게 좋아요.

---

## 임베딩 모델 선택

어떤 모델을 쓰느냐가 사실 가장 중요해요. 벡터 DB를 아무리 잘 골라도 임베딩 모델이 의미를 제대로 표현하지 못하면 검색이 실패하거든요.

MTEB(Massive Text Embedding Benchmark) 기준으로 주요 모델을 비교하면 이렇게 돼요:

| 모델 | MTEB 평균 | 한국어 | 차원 | 비용 |
| --- | --- | --- | --- | --- |
| [BGE-M3](https://huggingface.co/BAAI/bge-m3) | 66.1 | 우수 | 1,024 | 무료 |
| [text-embedding-3-large](https://platform.openai.com/docs/guides/embeddings) | 64.6 | 양호 | 3,072 | $0.13/1M |
| Cohere Embed v3 Multilingual | 64.0 | 우수 | 1,024 | $0.10/1M |
| [text-embedding-3-small](https://platform.openai.com/docs/guides/embeddings) | 62.3 | 양호 | 1,536 | $0.02/1M |
| [KoSimCSE-roberta](https://huggingface.co/BM-K/KoSimCSE-roberta) | 한국어 특화 | 최적 | 768 | 무료 |

한국어 RAG라면 BGE-M3 또는 KoSimCSE-roberta가 출발점이에요.

그리고 반드시 지켜야 할 원칙이 있어요.

```
❌ 잘못된 경우
인덱싱: BGE-M3 → 벡터 공간 A
검색:  OpenAI → 벡터 공간 B
→ 벡터 공간이 달라 의미 있는 검색 불가

✅ 올바른 경우
인덱싱: BGE-M3 → 벡터 공간 A
검색:  BGE-M3 → 동일 벡터 공간 A
→ 정확한 유사도 계산
```

**인덱싱과 검색에서 반드시 같은 모델을 써야 해요.** 당연해 보이지만 실제 코드에서 실수하는 경우가 꽤 많아요.

## 정리

| 항목 | 핵심 포인트 |
| --- | --- |
| 청킹 | 256~512 토큰, 10~20% overlap이 기본 |
| 구조화 문서 | Hierarchical chunking으로 헤더 구조 보존 |
| 메타데이터 | 자주 필터링할 것만, cardinality 낮게 |
| 임베딩 모델 | 한국어면 BGE-M3 or KoSimCSE |
| 모델 일관성 | 인덱싱과 검색에서 반드시 동일 모델 |

DB보다 전처리가 품질을 결정해요. 어떤 DB를 쓸지보다 어떻게 넣을지를 먼저 설계하는 게 맞아요.

다음 편에서는 실제 벡터 DB 6개의 벤치마크 숫자를 비교하면서 어떤 상황에 어떤 DB가 유리한지 살펴볼게요.
