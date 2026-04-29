# 11 — Connection Pooling

🔑 Postgres forks one OS process per connection — pooling (PgBouncer) is mandatory at scale.

Source: https://www.pgbouncer.org/usage.html

## Why Pool
- Each backend = ~10MB+ RSS, expensive to start.
- `max_connections` is bounded by RAM and locks; typical sweet spot is 100–300.
- Pooler multiplexes thousands of client conns onto a small server pool.

## PgBouncer Modes
| Mode | Server conn returned to pool when... | Client features lost |
|---|---|---|
| **session** | Client disconnects | None — full session compat |
| **transaction** | Tx commits/rolls back | Session GUCs, cursors, `LISTEN`, prepared stmts (mostly) |
| **statement** | Each statement ends | Multi-statement tx; rarely used |

💡 Most apps use **transaction pooling** — small server pool serves many short-lived transactions.

## Sizing Heuristics
- Server pool size ≈ `effective vCPU × 2..4`.
- `max_client_conn` can be huge (thousands).
- Aim for `pool_size + reserve_pool_size ≤ Postgres max_connections - admin headroom`.

## Prepared Statements Gotcha
- ⚠️ Transaction pooling re-uses server backends across clients → cached prepared statements in one client's plan cache aren't valid for the next.
- Fixes:
  - Disable client-side prepared stmts (asyncpg: `statement_cache_size=0`).
  - Use **PgBouncer 1.21+** with `server_prepared_statements = on` (tracks PSs across clients).
  - Or use **session pooling** (no multiplexing).

## SQLAlchemy 2.0 + asyncpg
```python
engine = create_async_engine(
    "postgresql+asyncpg://app@pgbouncer:6432/app",
    pool_size=10,           # SA pool, in front of PgBouncer
    max_overflow=20,
    pool_pre_ping=True,
    connect_args={"statement_cache_size": 0},  # tx-pooling safe
)
```
- ⚠️ SA's pool sits in front of PgBouncer; size it to your worker count, not your tenant count.

## Alternatives
- **pgcat** — Rust, multi-tenant, sharding, mirroring.
- **Odyssey** — multi-threaded.
- Server-side: PG 14+ has connection limits per role/db but no native pooler.

## Tags
[[PostgreSQL]] [[PgBouncer]] [[Connection Pooling]] [[asyncpg]]
