# 06 — Transactions and Isolation

🔑 Postgres defaults to READ COMMITTED, but Serializable (SSI) gets you a true snapshot with retry-on-conflict semantics.

Source: https://www.postgresql.org/docs/current/transaction-iso.html

## Isolation Levels
| Level | Sees | Anomalies prevented |
|---|---|---|
| READ UNCOMMITTED | Same as RC in PG | (no dirty reads anyway) |
| **READ COMMITTED** (default) | Latest committed at each statement | Dirty reads |
| **REPEATABLE READ** (snapshot) | One snapshot per tx | + Non-repeatable reads, phantom reads |
| **SERIALIZABLE** (SSI) | RR + serialization checks | + Write skew |

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ...
COMMIT;
```

⚠️ Under SERIALIZABLE, commits can fail with `40001 serialization_failure`. **Apps must retry.**

## Savepoints
```sql
BEGIN;
INSERT INTO ...;
SAVEPOINT s1;
INSERT INTO ...;       -- might fail
ROLLBACK TO SAVEPOINT s1;
COMMIT;
```

## Locking
- Row locks via `SELECT ... FOR UPDATE` / `FOR NO KEY UPDATE` / `FOR SHARE`.
- `SKIP LOCKED` — skip rows another tx holds (queue workers).
- `NOWAIT` — fail instead of block.

```sql
SELECT id FROM jobs
WHERE status = 'queued'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

## Advisory Locks
- App-defined locks keyed by integers, not tied to rows.
- Session-level (`pg_advisory_lock`) or tx-level (`pg_advisory_xact_lock`).
- 💡 Useful for "only one worker runs this job" without a row.

## Deadlocks
- Detected automatically; one tx is aborted with `40P01`.
- Prevent by acquiring locks in a consistent order.

💡 sqlalchemy 2.0 async: configure isolation per engine (`create_async_engine(url, isolation_level="REPEATABLE READ")`) or per connection (`conn.execution_options(isolation_level=...)`).

## Tags
[[PostgreSQL]] [[Transactions]] [[Isolation]] [[Locks]]
