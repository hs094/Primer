# 09 — Comparison Matrix

🔑 No single "best" vector DB — pick on deployment shape and adjacency to existing stack.

## Capability Matrix
| | [[pgvector]] | [[Qdrant]] | [[Weaviate]] | [[LanceDB]] | Pinecone | Chroma |
|---|---|---|---|---|---|---|
| **Deployment** | PG extension | Self-host / Cloud | Self-host / Cloud | Embedded / S3 | Managed only | Embedded / Server |
| **Scaling** | PG sharding (manual) | Native sharding + replicas | Sharding + replicas | Object-store horizontal | Auto (serverless) | Single node |
| **Hybrid search** | Manual (FTS + vector) | Built-in (sparse+dense+RRF) | Built-in (BM25+vector) | FTS + vector | Sparse-dense indexes | Built-in |
| **Metadata filter** | SQL `WHERE` | Rich JSON filter tree | GraphQL `where` | SQL `where` clause | `$eq` / `$in` JSON | `where` dict |
| **Embedded mode** | No (needs PG) | No | Limited (Python only) | **Yes** | No | **Yes** |
| **Managed cloud** | Many vendors | Qdrant Cloud | Weaviate Cloud | LanceDB Cloud | Pinecone (only mode) | Chroma Cloud |
| **License** | PostgreSQL | Apache 2.0 | BSD-3 | Apache 2.0 | Proprietary | Apache 2.0 |
| **Languages** | Any PG client | Py / TS / Rust / Go / Java | Py / TS / Java / Go | Py / TS / Rust / Java | Py / TS / Java / Go | Py / TS |
| **Async Python** | psycopg async | `AsyncQdrantClient` | v4 client (sync core) | `connect_async` | sync (v5) | `AsyncHttpClient` |

## Recommendations by Use Case
| Scenario | Pick |
|---|---|
| Already on Postgres, < 10M vectors | [[pgvector]] |
| Self-hosted RAG, hybrid search, payload filters | [[Qdrant]] |
| Want vectorizers + multi-tenancy out of box | [[Weaviate]] |
| Notebook / desktop app / S3-backed lake | [[LanceDB]] |
| "I refuse to run state" | Pinecone |
| Tiny prototype, single process | Chroma |
| > 1B vectors, custom infra team | Qdrant or Vespa/Milvus (out of scope) |

⚠️ Avoid premature migration. The cost of moving 100M embeddings between DBs (re-embed, re-index, dual-write window) usually dwarfs the marginal benefit of "the better DB".

💡 Most production stacks land on **one** of {pgvector, Qdrant, Pinecone} for the hot path, with [[LanceDB]] for offline/eval pipelines.

## Tags
[[VectorDB]] [[pgvector]] [[Qdrant]] [[Weaviate]] [[LanceDB]] [[RAG]]
