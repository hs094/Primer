# 13 — Full Text Search

🔑 Postgres has a built-in FTS engine: documents become `tsvector`, queries become `tsquery`, matched with `@@` and accelerated by GIN.

Source: https://www.postgresql.org/docs/current/textsearch.html

## Core Types
| Type | Holds |
|---|---|
| `tsvector` | Sorted list of lexemes + positions |
| `tsquery` | Parsed query (terms + operators) |

```sql
SELECT to_tsvector('english', 'The Quick brown foxes');
-- 'brown':3 'fox':4 'quick':2

SELECT to_tsvector('english', 'jumping foxes')
       @@ to_tsquery('english', 'fox & jump');
-- true
```

## Indexing
```sql
ALTER TABLE articles ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')),  'A') ||
    setweight(to_tsvector('english', coalesce(body,  '')),  'B')
  ) STORED;

CREATE INDEX articles_tsv_idx ON articles USING GIN (tsv);
```
- `setweight` lets `ts_rank` boost title hits over body.

## Query Builders
| Function | Behavior |
|---|---|
| `to_tsquery('cat & dog')` | Strict syntax |
| `plainto_tsquery('cat dog')` | AND of words |
| `phraseto_tsquery('big cat')` | Phrase match |
| `websearch_to_tsquery('"big cat" -kitten')` | Google-like |

💡 `websearch_to_tsquery` is what you want for user input — quotes, OR, negation are handled.

## Ranking
```sql
SELECT id, ts_rank(tsv, q) AS rank
FROM articles, websearch_to_tsquery('english', 'postgres index') q
WHERE tsv @@ q
ORDER BY rank DESC
LIMIT 20;
```
- `ts_rank` — frequency-based.
- `ts_rank_cd` — cover-density (closer matches rank higher).

## Language Config
- `regconfig` (`'english'`, `'simple'`, etc.) controls stemming + stopwords.
- ⚠️ Mismatched configs at index time vs query time → no matches. Pin both to the same.
- 💡 `'simple'` keeps tokens as-is — useful for usernames/codes.

## When to Reach for Something Else
- Multi-language ranking, typo tolerance, vector search → use [[pgvector]] + FTS hybrid, or external search (OpenSearch, Meilisearch).

## Tags
[[PostgreSQL]] [[Full Text Search]] [[GIN]] [[tsvector]]
