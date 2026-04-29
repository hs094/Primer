# 04 — Indexes

🔑 Match the index type to the operator — btree is default, but GIN/GiST/BRIN unlock arrays, jsonb, full-text, geo, and time-series.

Source: https://www.postgresql.org/docs/current/indexes-types.html

## Types
| Type | Best For | Operators |
|---|---|---|
| **btree** | Equality + range on scalars | `= < <= > >= BETWEEN ORDER BY` |
| **hash** | Equality only (large keys, big tables) | `=` |
| **GIN** | Composite values: arrays, jsonb, tsvector | `@> ? && @@` |
| **GiST** | Geo, ranges, nearest-neighbor | `&& <-> @>` |
| **SP-GiST** | Non-balanced (quad-trees, tries) | varied |
| **BRIN** | Huge, naturally-ordered tables (time-series) | range |

## Partial & Expression
```sql
-- Partial: index only what you query
CREATE INDEX active_users ON users (email) WHERE deleted_at IS NULL;

-- Expression: index a derived value
CREATE INDEX users_lower_email ON users (lower(email));
```

## Covering Indexes (INCLUDE)
```sql
CREATE INDEX orders_user_inc
  ON orders (user_id)
  INCLUDE (status, total);
```
- `INCLUDE` columns aren't part of the key — added as payload.
- Enables **index-only scans** when all needed columns live in the index + visibility map says page is all-visible.

## Index-Only Scan
- Requires the table page to be marked all-visible (set by VACUUM).
- ⚠️ Fresh writes → heap fetch falls back; run `VACUUM` to refresh visibility map.

## Build Modes
- `CREATE INDEX` — takes `ShareLock`, blocks writes.
- `CREATE INDEX CONCURRENTLY` — no write block; slower; can leave INVALID indexes if it fails.

💡 `EXPLAIN (ANALYZE, BUFFERS)` confirms which index is hit and whether a scan was index-only.

## Tags
[[PostgreSQL]] [[Indexes]] [[GIN]] [[btree]]
