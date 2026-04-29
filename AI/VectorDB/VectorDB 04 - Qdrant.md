# 04 — Qdrant

🔑 [[Qdrant]] is the "no-frills, payload-first" open-source vector DB. Rust core, gRPC + REST, hybrid + quantization built in.

Source: https://qdrant.tech/documentation/

## Mental Model
- **Collection** — schema for one set of vectors (named vectors supported)
- **Point** — `id` + `vector(s)` + `payload` (arbitrary JSON, indexable)
- **Filter** — boolean tree over payload, applied **inside** the ANN walk

## Async Python Client
```python
from qdrant_client import AsyncQdrantClient, models

client = AsyncQdrantClient(url="http://localhost:6333")
await client.create_collection(
    "docs",
    vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE),
)
await client.upsert("docs", points=[
    models.PointStruct(id=1, vector=vec, payload={"tenant": "acme", "lang": "en"}),
])
hits = await client.query_points(
    "docs", query=query_vec, limit=10,
    query_filter=models.Filter(must=[
        models.FieldCondition(key="tenant", match=models.MatchValue(value="acme"))
    ]),
).points
```

## Hybrid Search (Sparse + Dense + RRF)
```python
await client.query_points("docs",
    prefetch=[
        models.Prefetch(query=models.SparseVector(indices=[...], values=[...]),
                        using="sparse", limit=20),
        models.Prefetch(query=dense_vec, using="dense", limit=20),
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
)
```

## gRPC vs REST
REST is browser-friendly and fine ≤ a few k QPS. gRPC is 2–4× faster with smaller payloads and native streaming upserts. 💡 REST in services, gRPC in ingestion pipelines.

## Quantization
⚠️ Toggle on the collection: `quantization_config=models.ScalarQuantization(...)`. With `always_ram=True`, quantized vectors stay hot while originals spill to disk — same recall, ~4× memory cut.

## Tags
[[VectorDB]] [[Qdrant]] [[HNSW]] [[ANN]] [[RAG]]
