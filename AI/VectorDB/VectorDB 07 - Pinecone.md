# 07 — Pinecone

🔑 Pinecone is **fully managed only** — no self-host. You trade source-availability for not running a stateful service. Serverless tier is the current default.

Source: https://docs.pinecone.io/

## Concepts
- **Index** — analogous to a collection. Dense, sparse, or both.
- **Namespace** — logical partition inside an index; cheap multi-tenancy
- **Record** — `id` + `values` (vector) + `metadata` (filterable JSON)
- **Serverless** — pay per read/write/storage; no pod sizing
- **Pod-based** (legacy) — pick `p1` / `s1` / `p2`, pre-allocated capacity

## Python Client (v5+)
```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key=API_KEY)
if not pc.has_index("docs"):
    pc.create_index(
        name="docs",
        dimension=1536,
        metric="cosine",
        spec=ServerlessSpec(cloud="aws", region="us-east-1"),
    )
idx = pc.Index("docs")

idx.upsert(namespace="acme", vectors=[
    {"id": "doc-1", "values": vec, "metadata": {"lang": "en"}}
])
res = idx.query(
    namespace="acme",
    vector=query_vec, top_k=10,
    filter={"lang": {"$eq": "en"}},
    include_metadata=True,
)
```

## Sparse-Dense Hybrid
Pinecone supports a single record carrying both `values` (dense) and `sparse_values` (lexical, e.g. SPLADE). Query with `alpha` to weight them — `alpha=1.0` is pure dense, `0.0` is pure sparse.

## Tradeoffs
⚠️ Vendor lock-in is real: proprietary format, no export, region-priced. Pros: zero ops, serverless scales near-zero, inference + rerank built in. Cons: closed source, egress + per-read pricing surprises, fewer index knobs.

💡 Default to Pinecone when "another service to run" is the actual blocker. Otherwise [[Qdrant]] or [[pgvector]] gives the same primitives without the bill.

## Tags
[[VectorDB]] [[ANN]] [[RAG]]
