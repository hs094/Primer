# 02 — HNSW vs IVF

🔑 Two index families dominate: **[[HNSW]]** (graph, fast + greedy on RAM) and **IVF** (clusters, cheap + slower). Quantization is the orthogonal axis that shrinks both.

## HNSW — Hierarchical Navigable Small World
Builds a multi-layer proximity graph; queries greedily descend layers. Excellent recall/latency, no training step, but **vectors stay in RAM**.

| Param | Effect |
|---|---|
| `M` | Edges per node. Higher = better recall, more RAM (typical 16–64) |
| `ef_construction` | Build-time candidate list. Higher = better graph, slower build |
| `ef_search` | Query-time candidate list. The main recall/latency dial |

## IVF — Inverted File
k-means clusters vectors into `nlist` partitions; query probes `nprobe` nearest centroids.

| Param | Effect |
|---|---|
| `nlist` | Number of clusters. ≈ √N is a common starting point |
| `nprobe` | Clusters scanned per query. Higher = better recall, slower |

⚠️ IVF needs **training** on a representative sample before insert; HNSW does not.

## Quantization (Compression)
- **Scalar (SQ8)** — float32 → int8. ~4× smaller, ~1% recall loss. Cheap win.
- **Product (PQ)** — split vector into M sub-vectors, k-means each. 8–32× smaller, noticeable recall hit; usually paired with rerank on raw vectors.
- **Binary** — 1 bit per dim with Hamming distance. 32× smaller, only viable for some embedding models (e.g. matryoshka, Cohere binary-trained).

## Picking
| Need | Pick |
|---|---|
| < 10M vectors, latency-critical | HNSW |
| > 100M, RAM-bound | IVF + PQ |
| Cheapest "good enough" | IVF-Flat with high `nprobe` |
| Best recall, money no object | HNSW + SQ8 + rerank |

💡 [[Qdrant]] and [[pgvector]] both default to HNSW now. Reach for IVF when RAM is the bottleneck, not latency.

## Tags
[[VectorDB]] [[HNSW]] [[ANN]] [[Embeddings]]
