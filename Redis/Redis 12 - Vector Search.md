# 12 — Vector Search

🔑 Redis Stack stores embeddings as a `VECTOR` field on a HASH or JSON; `FT.CREATE` builds a FLAT or HNSW index; KNN/range queries via `FT.SEARCH`.

Source: https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/

## Index Algorithms
| Algorithm | Recall | Speed | Memory | Use |
|---|---|---|---|---|
| **FLAT** | Exact | O(N) per query | Lowest | Small (<100k) sets, ground-truth eval |
| **HNSW** | ~Approximate | Sub-linear | Higher (graph layers) | Production-scale ANN |

Distance metrics: `L2` (Euclidean), `IP` (inner product), `COSINE`.

## Create an Index
```
FT.CREATE docs ON HASH PREFIX 1 doc:
  SCHEMA
    text   TEXT
    embedding VECTOR HNSW 6
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
```
- `PREFIX 1 doc:` indexes any HASH whose key starts with `doc:`.
- HNSW knobs: `M` (graph degree), `EF_CONSTRUCTION`, `EF_RUNTIME` — tune recall vs latency.

## KNN Query
```
FT.SEARCH docs "*=>[KNN 5 @embedding $vec AS score]"
  PARAMS 2 vec <bytes>
  SORTBY score
  RETURN 2 text score
  DIALECT 2
```
- Vectors are passed as raw bytes (`np.float32(...).tobytes()`).
- Combine with filters: `"@tag:{python} =>[KNN 10 @embedding $vec]"`.

## RedisVL — Pythonic Layer
```python
from redisvl.index import AsyncSearchIndex
from redisvl.query import VectorQuery

idx = AsyncSearchIndex.from_yaml("schema.yaml", redis_url="redis://localhost:6379")
await idx.create(overwrite=False)

await idx.load([{"id": "doc:1", "text": "hi", "embedding": vec.tobytes()}])

q = VectorQuery(vector=qvec, vector_field_name="embedding", num_results=5,
                return_fields=["text"], return_score=True)
results = await idx.query(q)
```
- Schema YAML drives `FT.CREATE` — keep it in source control.
- Provides query builders (`VectorQuery`, `RangeQuery`, `FilterQuery`) and a vectorizer abstraction over OpenAI / HF / Cohere.

## ⚠️ Gotchas
- Need Redis Stack (or `RediSearch` module). Vanilla Redis OSS won't `FT.CREATE`.
- Match `DIM` to your embedding model exactly. Mismatch = silent garbage results.
- Reindexing 10M vectors with HNSW is slow — provision RAM + CPU.

## Tags
[[Redis]] [[Vector Search]] [[RedisVL]] [[HNSW]] [[Embeddings]]
