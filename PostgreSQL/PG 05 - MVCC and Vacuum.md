# 05 — MVCC and Vacuum

🔑 MVCC keeps old row versions until no snapshot needs them — vacuum is what eventually reclaims that space.

Source: https://www.postgresql.org/docs/current/mvcc.html

## How MVCC Works
- Every row (tuple) has hidden `xmin` (creating xid) and `xmax` (deleting/updating xid).
- A snapshot sees a tuple if `xmin` committed before snapshot start AND (`xmax` is null OR aborted OR after snapshot).
- `UPDATE` = new tuple version + old tuple gets `xmax`. **Updates aren't in-place.**
- Readers don't take row locks; writers don't block readers.

## Dead Tuples & Bloat
- Old versions = "dead tuples" until vacuumed.
- Bloat = tables/indexes growing past their live size.
- HOT (Heap-Only Tuple) updates: if no indexed column changed and there's room on the page, update doesn't touch indexes — much cheaper.

## VACUUM
| Variant | Effect |
|---|---|
| `VACUUM` | Marks dead space reusable in-place. Non-blocking. |
| `VACUUM FULL` | Rewrites table to a new file. **AccessExclusiveLock.** Reclaims to OS. |
| `VACUUM ANALYZE` | Vacuum + refresh planner stats. |
| `CLUSTER` | Like FULL, but orders rows by an index. Also exclusive. |

⚠️ Never run `VACUUM FULL` on a live OLTP table without a maintenance window — it locks out readers.

## Autovacuum
- Background workers triggered when dead tuples exceed `autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live`.
- Per-table tuning: `ALTER TABLE foo SET (autovacuum_vacuum_scale_factor = 0.05);`.

## Freeze & Wraparound
- `xid` is 32-bit. Old tuples must be "frozen" before the counter wraps.
- `autovacuum_freeze_max_age` triggers an aggressive freeze vacuum.
- ⚠️ Ignoring autovacuum on huge tables risks **transaction ID wraparound** — the DB will refuse new writes.

## Detecting Bloat
- `pgstattuple` extension, or query `pg_stat_user_tables.n_dead_tup`.
- 💡 `pg_stat_all_tables` shows `last_autovacuum`, `n_dead_tup`, `n_live_tup`.

## Tags
[[PostgreSQL]] [[MVCC]] [[Vacuum]] [[Bloat]]
