# 01 — Foundations

🔑 A vector database is two things glued together: a **vector index** for ANN search and a **store** for the original payload. Conflating them is where most "should I just use Postgres?" debates start.

## Embeddings as Coordinates
An [[Embeddings|embedding]] model maps a chunk of text/image/audio to a fixed-length float vector. Semantic similarity becomes geometric proximity — that's the only trick.

```python
from openai import AsyncOpenAI
client = AsyncOpenAI()
resp = await client.embeddings.create(model="text-embedding-3-small", input="hello")
vec = resp.data[0].embedding  # list[float], len 1536
```

## Similarity Metrics
| Metric | Formula | When |
|---|---|---|
| **Cosine** `<=>` | `1 − (a·b)/(‖a‖‖b‖)` | Default for normalized text embeddings |
| **Dot** `<#>` | `−(a·b)` | When magnitude carries signal (e.g. SPLADE) |
| **L2** `<->` | `‖a − b‖₂` | Image embeddings, geometry-style models |

💡 If your embeddings are L2-normalized, cosine and dot rank identically — pick whichever your DB makes faster.

## Exact vs Approximate NN
- **Exact** kNN scans every vector. O(N·d). Fine ≤ ~100k rows.
- **[[ANN]]** (HNSW, IVF, ScaNN) trades a few percent recall for 100×–1000× speedup. Above ~1M vectors, mandatory.

## Storage vs Index — Why Separate
⚠️ The index holds vectors + IDs only. The **payload** (text, metadata, source URL) lives in a row store. A query is: ANN → top-K IDs → fetch payload → optionally rerank/filter.

This split is why pgvector ([[pgvector]]) collapses both into Postgres, while [[Qdrant]]/[[Weaviate]]/[[LanceDB]] ship a tighter coupling but still log payload separately.

## Tags
[[VectorDB]] [[Embeddings]] [[ANN]] [[RAG]]
