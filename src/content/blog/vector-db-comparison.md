---
title: '벡터 DB 3편: 주요 솔루션 기술 비교 & 벤치마크'
description: 'Pinecone, Qdrant, Weaviate, Milvus, pgvector, Chroma — 6개 솔루션을 벤치마크 수치와 함께 비교해요. 필터링 성능, 하이브리드 검색, 멀티벡터 지원까지 실무 기준으로 정리했어요.'
pubDate: '2026-05-07'
heroImage: '../../assets/blog-placeholder-3.jpg'
tags: ['Vector DB', 'Qdrant', 'Pinecone', 'Weaviate', 'Milvus', 'pgvector', 'Benchmark']
---

벡터 DB를 고르려고 문서를 찾아보면 벤치마크 숫자들이 쏟아져요.

"QPS 1,420", "p99 9ms", "Recall@10 0.994" — 각 DB 공식 사이트마다 자기가 제일 빠르다고 하거든요. 어떤 숫자를 어떻게 봐야 하는지, 처음엔 막막했어요.

이번 편은 현재 실무에서 가장 많이 쓰이는 6개 솔루션을 같은 조건의 공개 벤치마크로 비교해요. 스펙표 말고 실제 선택 기준이 될 수치들로요.

## 비교 대상 솔루션

| 솔루션 | 카테고리 | 선정 이유 |
| --- | --- | --- |
| **Pinecone** | 클라우드 전용 | 가장 많이 쓰이는 관리형 벡터 DB |
| **Qdrant** | 오픈소스/클라우드 | 고성능, 풍부한 필터링, 활발한 커뮤니티 |
| **Weaviate** | 오픈소스/클라우드 | 하이브리드 검색, 멀티모달 지원 |
| **Milvus** | 오픈소스/Zilliz | 대규모 처리에 강한 오픈소스 |
| **pgvector** | RDB 확장 | PostgreSQL 기반, 기존 인프라 활용 |
| **Chroma** | 로컬/오픈소스 | 개발·프로토타입 표준 |

---

## 기술 스펙 비교

### 인덱스 알고리즘 지원

| 솔루션 | HNSW | IVF | FLAT | DiskANN | 커스텀 |
| --- | --- | --- | --- | --- | --- |
| Pinecone | ✅ (내부) | ✅ (내부) | ✅ | ❌ | ❌ |
| Qdrant | ✅ | ❌ | ✅ | ❌ | ❌ |
| Weaviate | ✅ | ❌ | ✅ | ❌ | ❌ |
| Milvus | ✅ | ✅ | ✅ | ✅ | ✅ (ScaNN, ANNOY) |
| pgvector | ✅ (v0.5+) | ✅ (IVFFlat) | ✅ | ❌ | ❌ |
| Chroma | ✅ (HNSW) | ❌ | ✅ | ❌ | ❌ |

Milvus가 가장 많은 인덱스를 지원해요. 데이터 특성에 맞는 알고리즘을 선택할 수 있어서 대규모 환경에서 유리하거든요.

### 하이브리드 검색 (벡터 + 키워드)

하이브리드 검색은 실무에서 생각보다 자주 필요해요. 정확한 키워드가 포함된 문서도 찾고, 의미적으로 유사한 문서도 동시에 찾아야 하는 경우가 많거든요.

| 솔루션 | 하이브리드 지원 | 방식 |
| --- | --- | --- |
| Pinecone | ✅ | Sparse + Dense |
| Qdrant | ✅ | Dense + Sparse (SPLADE) |
| Weaviate | ✅ | BM25 + Dense (내장) |
| Milvus | ✅ | Sparse + Dense (RRF, Weighted) |
| pgvector | 제한적 | SQL WHERE + 벡터 (수동 조합) |
| Chroma | ❌ | 지원 안 함 |

```python
# Weaviate 하이브리드 검색 — alpha로 벡터/키워드 비율 조절
results = collection.query.hybrid(
    query="E2E 테스트 flaky",
    alpha=0.7,     # 0=BM25만, 1=벡터만
    limit=5
)

# Qdrant 하이브리드 검색
from qdrant_client.models import NamedVector, NamedSparseVector, SearchRequest

results = client.search_batch(
    collection_name="docs",
    requests=[
        SearchRequest(vector=NamedVector("dense", dense_vec), limit=5),
        SearchRequest(vector=NamedSparseVector("sparse", sparse_vec), limit=5),
    ]
)
```

### 필터링 성능

필터링은 메타데이터 조건으로 검색 범위를 좁히는 거예요. DB마다 구현 방식이 달라 성능 차이가 커요. 특히 필터 조건이 복잡할수록 차이가 드러나거든요.

| 솔루션 | 필터링 방식 | 특징 |
| --- | --- | --- |
| Pinecone | Pre-filtering | 필터 후 벡터 검색 — 소규모에서 빠름 |
| **Qdrant** | Pre-filtering + HNSW graph filtering | 대규모에서도 정확하고 빠름 ★ |
| Weaviate | HNSW + inverted index 병행 | 하이브리드 필터링 효율적 |
| Milvus | Segment-level pre-filtering | 파티션으로 추가 최적화 가능 |
| pgvector | SQL WHERE (post-filtering) | 복잡한 조건 가능, 대규모에서 느림 |
| Chroma | Post-filtering | 단순하지만 대규모에서 비효율 |

---

## 공개 벤치마크

### ANN Benchmarks — 알고리즘 기준

[ANN Benchmarks](https://ann-benchmarks.com)는 알고리즘별 Recall vs QPS를 비교하는 표준 벤치마크예요.

**SIFT-1M 데이터셋 기준 (100만 벡터, 128차원)**

| 솔루션 | Recall@10 | QPS (단일 스레드) | 메모리 |
| --- | --- | --- | --- |
| Qdrant HNSW | 0.987 | 1,420 | 750MB |
| Milvus HNSW | 0.985 | 1,350 | 710MB |
| Weaviate HNSW | 0.981 | 1,180 | 820MB |
| Chroma HNSW | 0.974 | 980 | 860MB |
| Milvus IVF_FLAT | 0.969 | 2,100 | 520MB |
| pgvector HNSW | 0.961 | 890 | 980MB |

> Pinecone은 클라우드 전용으로 ANN Benchmarks에 직접 참여하지 않아요.

### VectorDBBench — 실제 DB 제품 비교

[VectorDBBench](https://zilliz.com/vector-database-benchmark-tool)는 클라우드 환경에서 실제 DB 제품 간 성능을 비교해요.

**Cohere 768차원, 100만 벡터, 필터 없음 기준**

| 솔루션 | QPS | p99 Latency | Recall@10 |
| --- | --- | --- | --- |
| Zilliz Cloud | 1,890 | 7ms | 0.996 |
| Pinecone (Standard) | 1,580 | 12ms | 0.997 |
| Qdrant Cloud | 1,420 | 9ms | 0.994 |
| Weaviate Cloud | 1,210 | 14ms | 0.987 |
| pgvector (RDS) | 680 | 28ms | 0.961 |

그런데 더 중요한 숫자가 있어요. **필터링이 들어갔을 때** 어떻게 변하느냐예요.

**필터링 포함 (50% 필터 조건) 기준**

| 솔루션 | QPS 변화 | Recall@10 변화 |
| --- | --- | --- |
| **Qdrant Cloud** | **-17%** | **-0.3% ★** |
| Pinecone | -44% | -1.6% |
| Weaviate Cloud | -35% | -1.2% |
| Zilliz Cloud | -34% | -0.8% |
| pgvector (RDS) | -69% | -3.7% |

Qdrant가 필터링 포함 시 QPS 저하(-17%)와 Recall 저하(-0.3%)가 가장 적어요. 반면 pgvector는 필터링이 들어가면 성능이 급격히 떨어지거든요.

---

## 솔루션별 한 줄 분석

### Pinecone — 빠른 시작의 기준

가장 많이 쓰이는 관리형 벡터 DB예요. 5분 안에 프로덕션 수준을 올릴 수 있거든요. 완전 관리형이라 인덱스 최적화와 스케일링이 자동으로 처리돼요.

다만 오픈소스가 아니라 벤더 종속이 생기고, 인덱스 알고리즘 커스터마이징이 불가해요. 데이터가 외부 서버에 저장된다는 점도 보안 규정이 있는 도메인에서는 고려해야 해요.

**적합한 상황**: 빠른 출시, 관리 오버헤드 최소화, 팀에 인프라 전문가 없을 때

### Qdrant — 필터링이 많다면

Rust 기반으로 메모리 효율이 높고, 필터링 성능이 벤치마크에서 가장 우수해요.

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

results = client.search(
    collection_name="docs",
    query_vector=embed("E2E 테스트"),
    query_filter=Filter(
        must=[
            FieldCondition(key="team", match=MatchValue(value="QA")),
            FieldCondition(key="created_at",
                         range=Range(gte="2026-01-01"))
        ]
    ),
    limit=10
)
```

온디스크 저장 옵션으로 RAM을 절약할 수 있고, 오픈소스와 클라우드 중 선택할 수 있어요.

**적합한 상황**: 복잡한 필터링이 많은 RAG, 비용 효율 중시, 오픈소스 선호

### Weaviate — 멀티모달 + 하이브리드 검색

BM25 + 벡터 하이브리드 검색이 내장되어 있고, 이미지 + 텍스트 멀티모달을 네이티브로 지원해요. Named Vectors로 필드별 다중 임베딩도 저장할 수 있어요.

```python
# 텍스트만 넣으면 자동으로 임베딩 생성 — OpenAI 모듈 연동
client.data_object.create(
    {"content": "E2E 테스트 가이드 내용..."},
    class_name="Document"
)

# near_text로 자연어 검색
result = (client.query
    .get("Document", ["content"])
    .with_near_text({"concepts": ["자동화 테스트"]})
    .with_limit(5)
    .do())
```

설정 복잡도가 높고 메모리 사용량이 상대적으로 많아요.

**적합한 상황**: 이미지+텍스트 멀티모달 RAG, 하이브리드 검색이 핵심

### Milvus — 수억 벡터 이상

업계 최대 규모의 벡터 처리를 지원해요. 가장 많은 인덱스 알고리즘을 지원하고, 파티셔닝으로 검색 공간을 분할할 수 있어요.

```python
# 팀별로 파티션을 나눠서 검색 공간 분리
collection.create_partition("QA_team")
collection.create_partition("FE_team")

results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 128}},
    limit=10,
    partition_names=["QA_team"]  # QA팀 문서만 검색
)
```

운영 복잡도가 높고 etcd, MinIO 등 의존성이 많아요. Zilliz Cloud로 관리형 전환은 가능해요.

**적합한 상황**: 수억 벡터 이상 대규모, 직접 인프라 운영 팀

### pgvector — PostgreSQL이 이미 있다면

기존 PostgreSQL 인프라를 그대로 활용할 수 있어요. SQL의 모든 기능과 벡터 검색을 조합할 수 있거든요.

```sql
SELECT d.id, d.content, d.team,
       d.embedding <-> $1 AS distance
FROM documents d
JOIN teams t ON d.team_id = t.id
WHERE t.name = 'QA'
  AND d.created_at > '2026-01-01'
  AND d.embedding <-> $1 < 0.5
ORDER BY distance
LIMIT 10;
```

벡터 수 100만 이상에서 성능이 급격히 저하되고, 필터링 포함 시 Recall도 많이 떨어져요.

**적합한 상황**: 이미 PostgreSQL 운영 중, 벡터 수 50만 이하, 복잡한 JOIN 조건 필요

### Chroma — 개발/프로토타입 표준

설치부터 첫 검색까지 5분이면 충분해요. LangChain, LlamaIndex와 완벽하게 통합돼요.

```python
import chromadb

client = chromadb.PersistentClient(path="./vectordb")
collection = client.get_or_create_collection("docs")

collection.add(
    documents=["E2E 테스트 내용...", "QA 프로세스..."],
    metadatas=[{"team": "QA"}, {"team": "QA"}],
    ids=["doc-1", "doc-2"]
)

results = collection.query(
    query_texts=["자동화 테스트 방법"],
    n_results=3,
    where={"team": "QA"}
)
```

프로덕션 대규모 사용에는 부적합하고, 하이브리드 검색을 지원하지 않아요.

**적합한 상황**: 로컬 개발, 프로토타입, 소규모 RAG

---

## 종합 비교표

| 항목 | Pinecone | Qdrant | Weaviate | Milvus | pgvector | Chroma |
| --- | --- | --- | --- | --- | --- | --- |
| 벡터 검색 속도 | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | ★★★ | ★★★ |
| 필터링 성능 | ★★★★ | ★★★★★ | ★★★★ | ★★★★ | ★★★ | ★★ |
| 하이브리드 검색 | ★★★★ | ★★★★ | ★★★★★ | ★★★★ | ★★ | ❌ |
| 멀티벡터 | ❌ | ★★★★ | ★★★★★ | ★★★★★ | ❌ | ❌ |
| 확장성 | ★★★★★ | ★★★★ | ★★★★ | ★★★★★ | ★★★ | ★★ |
| 운영 편의성 | ★★★★★ | ★★★★ | ★★★ | ★★ | ★★★★ | ★★★★★ |
| 오픈소스 | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 비용 효율 | ★★★ | ★★★★★ | ★★★★ | ★★★★ | ★★★★★ | ★★★★★ |

필터링이 많으면 Qdrant, 멀티모달이면 Weaviate, 대규모면 Milvus, 지금 당장 시작하려면 Pinecone 또는 Chroma예요.

다음 편에서는 "그래서 나는 뭘 써야 해?"를 의사결정 트리와 비용 시뮬레이션으로 직접 답할게요.

---

## 참고 자료

| 자료 | 내용 |
| --- | --- |
| [ANN Benchmarks](https://ann-benchmarks.com) | 알고리즘별 Recall–Latency 실측 비교 |
| [VectorDBBench (Zilliz)](https://zilliz.com/vector-database-benchmark-tool) | 클라우드 DB 제품 간 성능 비교 |
| [Qdrant 공식 벤치마크](https://qdrant.tech/benchmarks) | Qdrant 자체 측정 데이터 |
| [Weaviate 벤치마크](https://weaviate.io/developers/weaviate/benchmarks) | Weaviate 공식 성능 자료 |
