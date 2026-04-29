# 03 — ClickHouse Engines

🔑 The [[MergeTree]] family — pick the variant that matches your **collapsing/aggregation pattern**, because the merge does the work.

Source: https://clickhouse.com/docs/engines/table-engines/mergetree-family

## The Family
| Engine | What it does at merge | Typical use |
|---|---|---|
| `MergeTree` | Nothing — keeps all rows | Default, raw event log |
| `ReplacingMergeTree` | Keeps last row per `ORDER BY` (optionally by version col) | Upsert / dedupe by PK |
| `SummingMergeTree` | Sums numeric cols by `ORDER BY` | Pre-aggregated counters |
| `AggregatingMergeTree` | Merges `AggregateFunction` states | Materialized view target for `quantile`, `uniq`, `avg` |
| `CollapsingMergeTree` | Cancels +1 / -1 row pairs by `Sign` | Mutable state, e.g. session tracking |
| `VersionedCollapsingMergeTree` | Like above + version col | Out-of-order Sign rows |

## ReplacingMergeTree
```sql
CREATE TABLE users (
    user_id UInt64, email String, updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;
```
⚠️ Dedup happens **at merge time**, not insert. Query with `FINAL` (slow) or `argMax(email, updated_at) GROUP BY user_id` for guaranteed-correct reads.

## SummingMergeTree
```sql
ENGINE = SummingMergeTree
ORDER BY (date, country)
```
Numeric cols not in `ORDER BY` get summed when rows collide on the key. Still write `SUM() ... GROUP BY` in queries — merges aren't immediate.

## AggregatingMergeTree
Stores **partial aggregation states**. Pair with materialized views (Note 05).
```sql
CREATE TABLE daily_stats (
    day Date,
    uniq_users AggregateFunction(uniq, UInt64),
    p99_latency AggregateFunction(quantile(0.99), Float64)
) ENGINE = AggregatingMergeTree
ORDER BY day;
-- query: SELECT day, uniqMerge(uniq_users), quantileMerge(0.99)(p99_latency) ...
```

💡 Pick the engine first; let merges do reduction in the background instead of at query time.

## Tags
[[Clickhouse]] [[MergeTree]] [[ReplacingMergeTree]] [[AggregatingMergeTree]] [[SummingMergeTree]]
