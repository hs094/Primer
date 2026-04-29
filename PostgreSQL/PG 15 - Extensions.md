# 15 — Extensions

🔑 Extensions plug new types, functions, index AMs, and bgworkers into Postgres — `CREATE EXTENSION` is a one-line install.

Source: https://www.postgresql.org/docs/current/sql-createextension.html

## Lifecycle
```sql
SHOW shared_preload_libraries;            -- some extensions need preload
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
ALTER  EXTENSION pg_stat_statements UPDATE;
DROP   EXTENSION pg_stat_statements;
```
- Some require restart after editing `shared_preload_libraries` (pg_stat_statements, pg_cron, citus, timescaledb).
- Per-database; install in each DB you need it.

## Catalog
- `pg_available_extensions` — installed on disk.
- `pg_extension` — currently created in this DB.

## Heavyweight Picks
| Extension | What it does |
|---|---|
| **pg_stat_statements** | Aggregate query stats — top-N slow queries. Preload required. |
| **pg_cron** | Schedule SQL with cron syntax inside Postgres. |
| **PostGIS** | Geospatial types, indexes (GiST), functions. |
| **[[pgvector]]** | Vector type + ANN indexes (HNSW/IVFFlat). |
| **TimescaleDB** | Hypertables (auto-partitioned time-series), continuous aggregates. |
| **Citus** | Distributed/sharded Postgres across nodes. |

## Often-Useful Built-ins
| | |
|---|---|
| `pgcrypto` | `gen_random_uuid()`, hashing, encryption |
| `uuid-ossp` | Older UUID gens (v1, v3, v5) |
| `hstore` | Key-value strings (predates jsonb; legacy) |
| `pg_trgm` | Trigram similarity, GIN/GIST for `LIKE`/fuzzy |
| `btree_gin` / `btree_gist` | btree ops in GIN/GiST composite indexes |
| `tablefunc` | `crosstab` (pivot) |
| `unaccent` | Strip accents for FTS |

## Managed Service Caveats
- ⚠️ RDS / Cloud SQL / Supabase only allow extensions on their allowlist. Check before designing around one.
- 💡 Self-managed — read the extension's `make install` notes; many ship `.control` + `.sql` and need OS packages.

## Patterns
- 💡 Always preload `pg_stat_statements` in production. It costs almost nothing and pays back the first time something is slow.
- 💡 Combine `pg_trgm` with GIN for fast `ILIKE '%foo%'` and ranked fuzzy search.

## Tags
[[PostgreSQL]] [[Extensions]] [[pg_stat_statements]] [[pgvector]]
