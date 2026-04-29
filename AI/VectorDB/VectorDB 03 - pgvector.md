# 03 — pgvector

🔑 [[pgvector]] adds a `vector` type and ANN indexes to Postgres. The killer feature isn't the index — it's that your embeddings live in the same transaction as your domain rows.

Source: https://github.com/pgvector/pgvector

## Types
`vector(N)` (float32, ≤ 2,000 dims), `halfvec(N)` (float16, ≤ 4,000), `bit(N)`, `sparsevec` (SPLADE-style).

## Schema + Index
```sql
CREATE EXTENSION vector;
CREATE TABLE docs (
  id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT NOT NULL,
  body TEXT,
  embedding vector(1536)
);
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON docs (tenant_id);  -- for pre-filter
```

## Distance Operators
`<->` L2, `<#>` negative inner product, `<=>` cosine, `<+>` L1. Pair index op-class to operator: `vector_cosine_ops` for `<=>`.

## Async Python (psycopg + pgvector)
```python
import psycopg
from pgvector.psycopg import register_vector_async

async with await psycopg.AsyncConnection.connect(DSN) as conn:
    await register_vector_async(conn)
    rows = await (await conn.execute(
        "SELECT id, body FROM docs WHERE tenant_id = %s "
        "ORDER BY embedding <=> %s LIMIT 10",
        (tenant_id, query_vec),
    )).fetchall()
```

## Pre- vs Post-filter
⚠️ With an HNSW index, a `WHERE` clause is applied **after** the ANN scan — you can blow past `LIMIT` before your filter matches. Mitigations:
- Set `hnsw.iterative_scan = strict_order` (PG 0.8+)
- Partition by tenant, or use a partial index per tenant
- For high-cardinality filters, use IVF with the filter column indexed

💡 Postgres-first stacks should default to pgvector and only graduate to a dedicated DB when index size dwarfs working set RAM.

## Tags
[[VectorDB]] [[pgvector]] [[HNSW]] [[ANN]] [[RAG]]
