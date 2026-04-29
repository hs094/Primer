# 01 — Overview

🔑 PostgreSQL is an open-source ORDBMS that pairs SQL standards compliance with MVCC, WAL durability, and a deep extension system.

Source: https://www.postgresql.org/docs/current/intro-whatis.html

## What It Is
- **ORDBMS** — relational + object features (composite types, inheritance, custom types/operators).
- **ACID** via [[MVCC]] — readers never block writers, writers never block readers.
- **Crash-safe** via [[WAL]] (write-ahead log) — every change logged before data files mutate.
- **Process-per-connection** model (forks a backend per client) — drives the need for [[PgBouncer]].

## Core Pillars
| Pillar | Mechanism |
|---|---|
| Concurrency | MVCC (snapshots, xmin/xmax) |
| Durability | WAL + fsync, checkpoints |
| Extensibility | `CREATE EXTENSION`, custom types/ops/index AMs |
| Replication | Streaming (physical) + logical (pub/sub) |
| Isolation | RC (default), RR, Serializable (SSI) |

## Versioning
- Yearly major release (e.g., 16, 17). Major = new features + on-disk format changes.
- Minor releases (16.1, 16.2) — bug/security only, no on-disk changes, drop-in.
- Each major supported ~5 years.

⚠️ Major upgrades require `pg_upgrade` or logical replication — minor upgrades are restart-only.

## Why It's Picked
- **Extensions** — [[pgvector]], PostGIS, TimescaleDB, Citus, pg_stat_statements, pg_cron.
- **JSONB** — schemaless docs with proper indexing.
- **SQL surface** — CTEs, window functions, LATERAL, FILTER, GROUPING SETS, MERGE (15+).
- **Battle-tested** — 30+ years of development, conservative defaults.

💡 With sqlalchemy 2.0 async, use `postgresql+asyncpg://` URLs and `create_async_engine(...)` to get full async I/O against Postgres.

## Tags
[[PostgreSQL]] [[MVCC]] [[WAL]] [[Database]]
