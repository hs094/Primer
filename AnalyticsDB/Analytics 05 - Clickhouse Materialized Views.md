# 05 — ClickHouse Materialized Views

🔑 A ClickHouse MV is an **insert trigger** that runs your `SELECT` on each incoming block and writes the result to a target table. Compute shifts from query time → insert time.

Source: https://clickhouse.com/docs/materialized-view/incremental-materialized-view

## Two Flavors
- **Incremental MV** — runs per insert block. Default, what you want 95% of the time.
- **Refreshable MV** — periodically re-runs the full query (like Postgres). For dimension tables, low-frequency rebuilds.

## The Pattern (Incremental + AggregatingMergeTree)
```sql
-- 1. raw event table
CREATE TABLE events (ts DateTime, user_id UInt64, ...) ENGINE = MergeTree ORDER BY ts;

-- 2. target — pre-aggregated, by day
CREATE TABLE daily_uniq (
    day Date,
    uniq_users AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree ORDER BY day;

-- 3. the MV that wires them
CREATE MATERIALIZED VIEW daily_uniq_mv TO daily_uniq AS
SELECT toDate(ts) AS day, uniqState(user_id) AS uniq_users
FROM events GROUP BY day;
```
Query:
```sql
SELECT day, uniqMerge(uniq_users) FROM daily_uniq GROUP BY day;
```

## `TO target_table` vs no target
🔑 Always use `TO`. Without it, ClickHouse creates a hidden `.inner.*` table you can't manage.

## `POPULATE` Clause
`CREATE MATERIALIZED VIEW ... POPULATE AS SELECT ...` runs the SELECT once over existing data on creation.
⚠️ **Dangerous**: any inserts during POPULATE are **dropped** from the MV. Prefer: create empty MV → backfill manually with `INSERT INTO target SELECT ...`.

## Common Pitfalls
- **JOINs**: only the leftmost table triggers the MV. Right-side dimension changes won't propagate.
- **`GROUP BY` ≠ target `ORDER BY`**: misalignment causes the MergeTree to fail to merge rows.
- **`UNION ALL`** unsupported — make one MV per branch, all writing to same target.
- The MV runs on **the inserted block only** — global `ORDER BY` / window functions over all data won't work.

💡 Mental model: MV = "cron of `INSERT INTO target SELECT ... FROM new_block`."

## Tags
[[Clickhouse]] [[MaterializedView]] [[AggregatingMergeTree]] [[Incremental]]
