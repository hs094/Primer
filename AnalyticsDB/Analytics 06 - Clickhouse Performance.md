# 06 — ClickHouse Performance

🔑 Performance in ClickHouse is **mostly schema design**, not tuning knobs. Get the primary key, low-cardinality typing, and write batching right; everything else is marginal.

Source: https://clickhouse.com/docs/optimize

## 1. Primary Key (`ORDER BY`)
- Pick columns matching **your most common WHERE prefix**. `ORDER BY (tenant_id, ts)` not `(ts, tenant_id)` if you always filter by tenant.
- Sparse index → only the leftmost columns help skipping.
- Cardinality matters: low-card cols first (e.g. `country`), then high-card (`user_id`).

## 2. `LowCardinality(String)`
Dictionary-encodes a column.
```sql
event_type LowCardinality(String),
country    LowCardinality(FixedString(2))
```
- 🔑 Use for any string column with **<10K distinct values** (status, country, plan).
- Smaller storage, faster `GROUP BY`, faster filters.
- ⚠️ Don't wrap >100K-cardinality strings — perf gets *worse*.

## 3. Data Skipping Indexes
Secondary indexes on **granules**, not rows. Skip whole blocks where no match is possible.
```sql
ALTER TABLE events
ADD INDEX idx_payload payload TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 4;
```
| Type | Use |
|---|---|
| `minmax` | Loosely-sorted numeric/date col |
| `set(N)` | Low-cardinality categoricals |
| `bloom_filter` / `tokenbf_v1` | String contains / equality |
| `text` (v24.8+) | Full-text inverted index |

⚠️ Skip indexes only help when correlated with the primary key. Otherwise they read but skip nothing.

## 4. Projections
Per-table alternate sort orders / pre-aggregations, transparent to queries:
```sql
ALTER TABLE events ADD PROJECTION p_by_user (
    SELECT * ORDER BY user_id, ts
);
```
Optimizer picks the right one. Like covering indexes, but columnar.

## 5. Async Inserts
🔑 Lots of small inserts kill MergeTree (each = a part). Either **batch client-side** or enable async:
```sql
SET async_insert = 1, wait_for_async_insert = 1;
INSERT INTO events VALUES (...);
```
Server buffers up to 100 MiB / 50–200 ms / 450 queries, then flushes one part. `wait_for_async_insert=1` keeps durability semantics.

## Diagnostics
- `EXPLAIN PLAN`, `EXPLAIN PIPELINE`
- `system.query_log`, `system.parts`, `system.merges`

## Tags
[[Clickhouse]] [[Performance]] [[LowCardinality]] [[Projections]] [[AsyncInserts]]
