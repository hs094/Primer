# 10 — Performance and EXPLAIN

🔑 `EXPLAIN (ANALYZE, BUFFERS)` is the truth — read plans bottom-up, mind row-estimate vs actual, watch buffer hits.

Source: https://www.postgresql.org/docs/current/using-explain.html

## EXPLAIN Anatomy
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT * FROM orders WHERE user_id = $1;
```
- `cost=startup..total` — planner estimate (units of seq page reads).
- `rows=` — estimated; vs `actual rows=` — what ran.
- `BUFFERS` — `shared hit/read/dirtied` shows cache behavior.

⚠️ Big gap between estimated and actual rows ⇒ bad plan. Run `ANALYZE` to refresh stats; raise `default_statistics_target` for skewed columns.

## Scan Types
| Scan | When |
|---|---|
| **Seq Scan** | Small tables, no usable index, fetching most rows |
| **Index Scan** | Selective predicate hits a btree |
| **Index Only Scan** | All needed columns in index + visibility map clean |
| **Bitmap Heap Scan** | Multiple indexes / mid-selectivity — builds a bitmap of TIDs |

## Join Types
| Join | Best For |
|---|---|
| **Nested Loop** | Tiny outer × indexed inner |
| **Hash Join** | Big × big, equi-join, fits in `work_mem` |
| **Merge Join** | Both sides sorted on join key |

💡 If a hash join spills, raise `work_mem` for that session: `SET LOCAL work_mem = '256MB';`.

## Planner Inputs
- `pg_statistic` — populated by `ANALYZE` (and autovacuum's analyze).
- Extended statistics: `CREATE STATISTICS` for correlated columns.
- `random_page_cost`, `effective_cache_size` — tune for SSDs (`random_page_cost = 1.1`).

## pg_stat_statements
```sql
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```
- Aggregated per normalized query — your top-N hotlist.

## Workflow
1. Identify slow query (pg_stat_statements / app traces).
2. `EXPLAIN (ANALYZE, BUFFERS)`.
3. Check estimates → run `ANALYZE` if off.
4. Add/adjust index; re-run; compare buffers.

## Tags
[[PostgreSQL]] [[EXPLAIN]] [[Performance]] [[Indexes]]
