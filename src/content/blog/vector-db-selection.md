---
title: '벡터 DB 4편: 내 상황에 맞는 DB 선택 가이드'
description: '의사결정 트리, 규모별 추천, 비용 시뮬레이션까지 — "그래서 나는 뭘 써야 해?"에 직접 답하는 가이드예요. 벡터 DB 시리즈 마지막 편이에요.'
pubDate: '2026-05-08'
heroImage: '../../assets/blog-placeholder-4.jpg'
tags: ['Vector DB', 'RAG', 'Pinecone', 'Qdrant', 'Chroma', 'pgvector', 'AI-Ready']
---

세 편을 다 읽고 나서도 "그래서 나는 뭘 써야 해?"라는 질문이 남는 분들이 있을 거예요.

알고리즘, 벤치마크, 기술 스펙 — 알면 알수록 선택이 더 어렵게 느껴지거든요. 이번 편은 그 질문에 직접 답해요. 의사결정 트리로 내 상황에 맞는 DB를 5분 안에 고르고, 비용 시뮬레이션으로 예산까지 가늠해볼 수 있어요.

## 의사결정 트리

아래 질문에 순서대로 답하면 돼요.

```
Q1. 지금 단계가 프로토타입/개발인가?
  YES → Chroma (로컬, 5분 설치)
  NO  → Q2로

Q2. 기존에 PostgreSQL을 운영 중인가?
  YES + 벡터 수 50만 이하 → pgvector (인프라 추가 불필요)
  YES + 벡터 수 50만 초과 → Q3로
  NO  → Q3로

Q3. 이미지 + 텍스트 멀티모달이 필요한가?
  YES → Weaviate
  NO  → Q4로

Q4. 벡터 수가 1억 개를 초과하는가?
  YES → Milvus / Zilliz Cloud
  NO  → Q5로

Q5. 복잡한 필터링이 검색의 핵심인가?
  YES → Qdrant
  NO  → Q6로

Q6. 인프라 운영 리소스가 없는가?
  YES → Pinecone (완전 관리형)
  NO  → Qdrant (오픈소스, 비용 효율)
```

대부분은 Q5~Q6에서 결정돼요. **필터링이 많으면 Qdrant, 관리형이 필요하면 Pinecone**이 기본값이에요.

---

## 규모별 선택 가이드

### 소규모 (벡터 수 ~50만, 월 쿼리 ~100만)

| 조건 | 추천 | 이유 |
| --- | --- | --- |
| 빠른 시작, 관리 최소 | Pinecone Starter | 무료, 설정 없음 |
| 기존 PostgreSQL 있음 | pgvector + Supabase Free | 인프라 추가 불필요 |
| 오픈소스 선호 | Qdrant Cloud Free | 1GB 무료, 성능 우수 |
| 로컬 개발 | Chroma | 설치 즉시 사용 |

**예상 비용**: 무료 ~ $25/월

사내 문서 RAG 정도면 대부분 무료 티어로 충분히 운영할 수 있어요. Confluence 5,000페이지를 청킹해도 벡터 수가 10만 개 수준이거든요.

### 중규모 (벡터 수 50만~1,000만, 월 쿼리 ~1,000만)

| 조건 | 추천 | 예상 비용 |
| --- | --- | --- |
| 관리형, 안정성 우선 | Pinecone Standard | ~$70~$150/월 |
| 필터링 많음, 비용 효율 | Qdrant Cloud | ~$25~$100/월 |
| 하이브리드 검색 핵심 | Weaviate Cloud | ~$25~$100/월 |
| 기존 PostgreSQL | pgvector Pro | $25~$100/월 |

**예상 비용**: $25~$300/월

### 대규모 (벡터 수 1,000만~1억)

| 조건 | 추천 | 예상 비용 |
| --- | --- | --- |
| AWS 기반 | AWS OpenSearch k-NN | $200~$500/월 |
| Azure 기반 | Azure AI Search | $250~$500/월 |
| GCP 기반 | Vertex AI Vector Search | $300~+/월 |
| 클라우드 중립 | Zilliz Cloud Dedicated | $200~+/월 |
| 자체 인프라 운영 | Milvus (셀프호스팅) | 인프라 비용만 |

### 초대규모 (벡터 수 1억 초과)

클라우드 네이티브(AWS OpenSearch, Azure AI Search, GCP Vertex AI)나 Milvus 분산 클러스터가 현실적인 선택이에요. 이 규모에서는 별도 POC와 계약 협의가 필요하거든요.

---

## 인프라/클라우드별 선택

현재 어떤 클라우드를 쓰느냐도 중요한 기준이에요.

| 현재 인프라 | 추천 솔루션 | 이유 |
| --- | --- | --- |
| AWS 기반 | OpenSearch k-NN or Pinecone | AWS 완전 통합, VPC 내 보안 |
| Azure 기반 | Azure AI Search | Azure OpenAI와 네이티브 연동 |
| GCP 기반 | Vertex AI Vector Search | BigQuery, Vertex AI 파이프라인 연동 |
| 멀티클라우드 | Pinecone or Qdrant Cloud | 클라우드 중립적 |
| 온프레미스 | Milvus or Qdrant (셀프호스팅) | 외부 데이터 유출 없음 |
| 서버리스 선호 | Pinecone Serverless or Zilliz Serverless | 사용량 기반 과금 |

---

## 비용 시뮬레이션

숫자로 직접 보면 더 명확해요.

### 시나리오 A — 사내 문서 RAG (소규모)

```
문서: Confluence 5,000페이지
평균 청크: 페이지당 20개 = 총 10만 벡터
차원: 1,536 (text-embedding-3-small)
월 쿼리: 10만 회

임베딩 생성 비용 (최초 1회):
  10만 청크 × 500토큰 = 5,000만 토큰
  → $0.02/1M × 50 = $1 (약 1,300원)

벡터 DB 월 비용:
  Pinecone Starter: 무료 (10만 << 200만 한도)
  pgvector Supabase Free: 무료
  Qdrant Cloud Free: 무료

→ 월 비용 $0 (임베딩 비용만 최초 $1)
```

많은 팀이 사내 RAG를 비용 이슈로 망설이는데, 소규모에서는 거의 무료예요.

### 시나리오 B — 중규모 서비스 RAG

```
문서: 100만 개 청크
차원: 1,536
월 쿼리: 1,000만 회

월 비용 추정:
  Pinecone Standard: ~$100~$200/월
  Qdrant Cloud:      ~$50~$100/월
  Weaviate Cloud:    ~$50~$100/월
  pgvector RDS:      ~$150/월 (DB 인스턴스 포함)
```

### 시나리오 C — 대규모 프로덕션

```
문서: 5,000만 개 청크
차원: 1,536
월 쿼리: 1억 회

월 비용 추정:
  Pinecone Enterprise:    ~$2,000+/월 (협의)
  Zilliz Cloud Dedicated: ~$800~$2,000/월
  Milvus 셀프호스팅:       서버 비용 ~$500~$1,000/월
  AWS OpenSearch:          ~$500~$2,000/월
```

---

## 단계별 전환 전략

처음부터 완벽한 DB를 고를 필요 없어요. 단계적으로 전환하는 게 현실적이에요.

```
Phase 1 (PoC, 1~2주)
  Chroma (로컬) + text-embedding-3-small
  → 개념 검증, 청킹 전략 실험, 비용 없음

Phase 2 (스테이징, 1~2개월)
  Pinecone Starter or Qdrant Cloud Free
  → 실제 데이터로 품질 측정, 무료로 운영

Phase 3 (프로덕션, 안정화 후)
  요구사항에 맞는 유료 플랜으로 전환
  → 측정된 성능 데이터 기반으로 선택

Phase 4 (스케일업, 필요 시)
  Milvus 셀프호스팅 or 엔터프라이즈 플랜
  → 규모가 검증된 후 전환
```

이 순서가 중요한 이유는, Phase 1~2를 거치면서 내 데이터에서 Recall이 어느 수준인지, 어떤 청킹 전략이 효과적인지 실측 데이터가 쌓이거든요. 그 데이터를 갖고 Phase 3 결정을 내리는 게 훨씬 정확해요.

---

## 최종 추천 요약

| 내 상황 | 추천 | 월 예상 비용 |
| --- | --- | --- |
| 지금 당장 테스트해보고 싶다 | **Chroma** (로컬) | 무료 |
| 소규모 RAG, 빠른 시작 | **Pinecone Starter** | 무료 |
| PostgreSQL 이미 있음 | **Supabase pgvector** | 무료~$25 |
| 필터링 많고 비용 효율 | **Qdrant Cloud** | $25~ |
| 이미지+텍스트 함께 검색 | **Weaviate** | $25~ |
| 1억 벡터 이상 대규모 | **Milvus / Zilliz** | $500~ |
| AWS/Azure/GCP 엔터프라이즈 | **각 클라우드 네이티브** | $200~ |

---

## 시리즈를 마치며

4편에 걸쳐 다룬 내용을 한 줄씩 정리하면 이렇게 돼요.

| 편 | 핵심 메시지 |
| --- | --- |
| **1편** | 벡터 DB는 의미를 저장한다. Recall-Latency tradeoff를 이해하라 |
| **2편** | DB보다 청킹과 임베딩 전략이 품질을 결정한다 |
| **3편** | 필터링 많으면 Qdrant, 멀티모달이면 Weaviate, 대규모면 Milvus |
| **4편** | 처음엔 Chroma로 시작해서 데이터가 쌓이면 마이그레이션하라 |

가장 좋은 DB는 지금 당장 시작할 수 있는 DB예요.

완벽한 선택을 기다리다 시작을 못 하는 것보다, Chroma로 시작해서 배우는 게 나아요. RAG 시스템에서 진짜 시간이 걸리는 건 DB 선택이 아니라 청킹 전략을 데이터에 맞게 튜닝하는 과정이거든요. 그건 어떤 DB를 고르든 직접 해봐야 알 수 있어요.
