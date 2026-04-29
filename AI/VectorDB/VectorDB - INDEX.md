# VectorDB — INDEX

🔑 Pack covers the **why** (foundations, indexes), the **what** (six DBs), and the **how** (hybrid + rerank). Read 01–02 first; pick DB notes by what you actually use.

## Foundations
- [[VectorDB 01 - Foundations]] — embeddings, similarity metrics, ANN vs exact, storage/index split
- [[VectorDB 02 - HNSW vs IVF]] — graph vs cluster indexes, quantization (SQ/PQ/binary)

## Per-Database
- [[VectorDB 03 - pgvector]] — Postgres extension; `vector(N)`, hnsw/ivfflat, distance ops
- [[VectorDB 04 - Qdrant]] — Rust core, payload filters, hybrid + quantization, async client
- [[VectorDB 05 - Weaviate]] — schema-first, vectorizer modules, multi-tenancy, GraphQL
- [[VectorDB 06 - LanceDB]] — embedded, columnar, S3-backed, async Python
- [[VectorDB 07 - Pinecone]] — managed-only, namespaces, sparse-dense hybrid
- [[VectorDB 08 - Chroma]] — minimal API, embedded → server with one line

## Comparison + Patterns
- [[VectorDB 09 - Comparison Matrix]] — capability table + recommendations by use case
- [[VectorDB 10 - Hybrid Search]] — dense + sparse, BM25, RRF fusion
- [[VectorDB 11 - Reranking]] — bi- vs cross-encoder cascade, Cohere + BGE

## Picking a DB — One-Line Heuristic
| Situation | Default |
|---|---|
| Already on Postgres | [[pgvector]] |
| Self-hosted, want all the knobs | [[Qdrant]] |
| Want vectorizers + multi-tenancy built in | [[Weaviate]] |
| Embedded / notebook / S3 lake | [[LanceDB]] |
| Refuse to run stateful infra | Pinecone |
| Tiny prototype | Chroma |

💡 The DB is rarely the bottleneck. Embedding model choice + chunking + reranking move the [[RAG]] quality needle far more than picking between Qdrant and Weaviate.

🧪 Build order for a new RAG system: pgvector or Chroma → measure recall@k → switch only if index size or QPS forces it → add hybrid → add rerank.

## Tags
[[VectorDB]] [[Embeddings]] [[ANN]] [[HNSW]] [[pgvector]] [[Qdrant]] [[Weaviate]] [[LanceDB]] [[RAG]]
