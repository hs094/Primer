# 02 — Data Types

🔑 Pick the narrowest correct type — Postgres has rich built-ins; misuse causes bloat, casting headaches, and timezone bugs.

Source: https://www.postgresql.org/docs/current/datatype.html

## Strings
| Type | Notes |
|---|---|
| `text` | Variable, unlimited. **Default choice.** |
| `varchar(n)` | Same perf as `text`; `n` is a check constraint. |
| `char(n)` | Blank-padded fixed. Avoid. |

⚠️ `varchar(n)` vs `text` is **not** a performance choice in Postgres — only a length cap.

## Numerics
| Type | Use |
|---|---|
| `smallint`/`integer`/`bigint` | Whole numbers. |
| `numeric(p,s)` / `decimal` | Exact, money, anything financial. |
| `real`/`double precision` | IEEE float — never for money. |
| `serial`/`bigserial` | Legacy; prefer `GENERATED ALWAYS AS IDENTITY` (10+). |

## Time
- `timestamptz` — stored as UTC, displayed in session TZ. **Use this.**
- `timestamp` — naive, no TZ info. Bug magnet across services.
- `date`, `time`, `interval`.

```sql
CREATE TABLE event (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  at    timestamptz NOT NULL DEFAULT now(),
  tags  text[] NOT NULL DEFAULT '{}',
  span  int4range
);
```

## Composite & Collection
- **Arrays** — `text[]`, `int[]`. GIN-indexable, `@>`, `&&`, `ANY`.
- **Range types** — `int4range`, `tstzrange`. Exclusion constraints for non-overlap.
- **ENUM** — ordered, type-safe; ⚠️ adding a value needs `ALTER TYPE` and can't be done in some old versions inside a tx.
- **UUID** — `uuid` type (16 bytes); `gen_random_uuid()` from `pgcrypto`.
- **Composite types** — row-as-value via `CREATE TYPE foo AS (...)`.

💡 In sqlalchemy 2.0: `Mapped[datetime]` with `DateTime(timezone=True)` maps cleanly to `timestamptz`; use `ARRAY(String)` for `text[]`.

## Tags
[[PostgreSQL]] [[Data Types]] [[UUID]] [[Timestamps]]
