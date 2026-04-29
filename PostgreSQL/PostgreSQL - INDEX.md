# PostgreSQL — INDEX

🔑 Master index for the PostgreSQL deep-dive pack: engine internals, query/perf, replication, security, search, vectors.

Source: https://www.postgresql.org/docs/current/

## Foundations
- [[PG 01 - Overview]] — ORDBMS, MVCC, WAL, extensions, versioning.
- [[PG 02 - Data Types]] — text, numeric, timestamptz, arrays, ranges, ENUM, UUID.
- [[PG 03 - JSONB]] — jsonb vs json, operators, GIN indexes, paths.
- [[PG 04 - Indexes]] — btree/hash/GIN/GiST/BRIN/SP-GiST, partial, INCLUDE, IOS.

## Engine
- [[PG 05 - MVCC and Vacuum]] — xmin/xmax, dead tuples, autovacuum, freeze, bloat.
- [[PG 06 - Transactions and Isolation]] — RC/RR/SSI, savepoints, advisory locks.
- [[PG 07 - Partitioning]] — range/list/hash, pruning, partition-wise joins.

## Replication & HA
- [[PG 08 - Logical Replication]] — pub/sub, slots, REPLICA IDENTITY, CDC.
- [[PG 09 - Streaming Replication and HA]] — WAL shipping, sync vs async, Patroni.

## Performance
- [[PG 10 - Performance and EXPLAIN]] — EXPLAIN ANALYZE, scans/joins, pg_stat_statements.
- [[PG 11 - Connection Pooling]] — PgBouncer modes, prepared statement gotcha.

## Security & Search
- [[PG 12 - Row Level Security]] — policies, USING vs WITH CHECK, multi-tenant.
- [[PG 13 - Full Text Search]] — tsvector/tsquery, GIN, ts_rank, websearch.

## Vectors & Ecosystem
- [[PG 14 - pgvector]] — vector(N), HNSW vs IVFFlat, ef_search/probes.
- [[PG 15 - Extensions]] — pg_stat_statements, pg_cron, PostGIS, pgvector, Timescale, Citus.
- [[PG 16 - Backup and PITR]] — pg_dump, pg_basebackup, WAL archive, recovery_target_time.

## Cross-links
- [[Database]] · [[MVCC]] · [[WAL]] · [[JSONB]] · [[pgvector]] · [[RLS]] · [[PgBouncer]] · [[Logical Replication]]

## Tags
[[PostgreSQL]] [[INDEX]]
