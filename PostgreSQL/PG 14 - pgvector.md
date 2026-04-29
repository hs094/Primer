# 14 — pgvector

🔑 `pgvector` adds a `vector(N)` type and ANN indexes (IVFFlat, HNSW) — embeddings live next to your relational data.

Source: https://github.com/pgvector/pgvector

## Setup
```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
  id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  content text,
  embedding vector(1536)        -- e.g. text-embedding-3-small
);
```

## Distance Operators
| Op | Meaning | Index opclass |
|---|---|---|
| `<->` | L2 distance | `vector_l2_ops` |
| `<#>` | Negative inner product | `vector_ip_ops` |
| `<=>` | Cosine distance | `vector_cosine_ops` |
| `<+>` | L1 (Manhattan) | `vector_l1_ops` |

⚠️ `<#>` returns **negative** inner product so smaller = closer (matches ORDER BY semantics).

## Indexes
| Index | Build | Recall | Notes |
|---|---|---|---|
| **IVFFlat** | Fast | Good | Needs `lists`; train on data |
| **HNSW** | Slow, big | Best | No training; `m`, `ef_construction` |

```sql
-- HNSW (recommended default)
CREATE INDEX ON documents
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- IVFFlat
CREATE INDEX ON documents
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);   -- ~ sqrt(N) for N rows
```

## Query-Time Tuning
```sql
SET hnsw.ef_search = 100;     -- higher = more accurate, slower
SET ivfflat.probes = 10;      -- # lists scanned

SELECT id, content
FROM documents
ORDER BY embedding <=> $1
LIMIT 10;
```
| Knob | Index | Trade-off |
|---|---|---|
| `hnsw.ef_search` | HNSW | recall ↑ / latency ↑ |
| `ivfflat.probes` | IVFFlat | recall ↑ / latency ↑ |
| `lists` | IVFFlat | build-time partition count |
| `m`, `ef_construction` | HNSW | graph density / build cost |

## Patterns
- **Hybrid search** — combine `<=>` with FTS (`tsvector @@ tsquery`) using `RRF` or weighted sums.
- **Filter + ANN** — pre-filter on `WHERE tenant_id = $1`; partial indexes per tenant if cardinality is low.
- ⚠️ ANN indexes are **approximate** — exact recall needs `SET enable_indexscan = off` or no index.

## sqlalchemy 2.0
```python
from pgvector.sqlalchemy import Vector

class Document(Base):
    __tablename__ = "documents"
    id: Mapped[int] = mapped_column(primary_key=True)
    embedding: Mapped[list[float]] = mapped_column(Vector(1536))
```

## Tags
[[PostgreSQL]] [[pgvector]] [[Vector Search]] [[Embeddings]]
