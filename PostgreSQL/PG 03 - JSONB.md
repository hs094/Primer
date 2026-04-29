# 03 — JSONB

🔑 `jsonb` is binary, indexed, and de-duplicated — use it for schemaless data, not `json`.

Source: https://www.postgresql.org/docs/current/datatype-json.html

## json vs jsonb
| | `json` | `jsonb` |
|---|---|---|
| Storage | Text, as-is | Decomposed binary |
| Whitespace/key order | Preserved | Lost |
| Duplicate keys | Kept | Last wins |
| Indexable | No (functional only) | **GIN** |
| Operator perf | Re-parse each call | Fast |

Use `jsonb` unless you need the original text byte-for-byte.

## Operators
| Op | Meaning |
|---|---|
| `->` | Get field as `jsonb` |
| `->>` | Get field as `text` |
| `#>` / `#>>` | Path get (jsonb / text) |
| `@>` | Contains |
| `<@` | Contained by |
| `?` | Has key |
| `?\|` | Has any key |
| `?&` | Has all keys |

```sql
SELECT data->>'email'
FROM users
WHERE data @> '{"role":"admin"}';
```

## Indexing
```sql
-- Default GIN: every key + value, supports @> ? ?| ?&
CREATE INDEX users_data_gin ON users USING gin (data);

-- jsonb_path_ops: smaller, faster, only @>
CREATE INDEX users_data_pathops ON users USING gin (data jsonb_path_ops);

-- Expression index for hot field
CREATE INDEX users_email ON users ((data->>'email'));
```

## SQL/JSON Path
```sql
SELECT jsonb_path_query(data, '$.items[*] ? (@.qty > 0).sku')
FROM orders;
```

## Performance vs Columns
- ⚠️ `jsonb` field access is slower than a real column. Promote hot keys to columns.
- Whole-doc updates rewrite the entire `jsonb` (no in-place patch on TOAST).
- 💡 Hybrid pattern: real columns for queryable/indexed fields, `jsonb` for the long tail.

## Tags
[[PostgreSQL]] [[JSONB]] [[GIN]] [[Indexes]]
