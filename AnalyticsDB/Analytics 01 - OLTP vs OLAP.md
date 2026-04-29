# 01 — OLTP vs OLAP

🔑 **Row stores** ([[PostgreSQL]], MySQL) optimize for transactions — read/write whole rows fast. **[[Columnar]]** stores ([[Clickhouse]], [[DuckDB]], BigQuery) optimize for analytics — scan a few columns over billions of rows.

## The Workloads
| Dim | OLTP | [[OLAP]] |
|---|---|---|
| Query | Point lookup, single row update | Aggregate millions of rows |
| Cols touched | Many (whole row) | Few (`SUM(amount)`) |
| Write pattern | High freq, tiny | Bulk loads, append |
| Latency target | <10 ms | sub-second to minutes |
| Example | "Get user 42's cart" | "Revenue by country last quarter" |

## Storage Layout
**Row layout** — values for one row are physically contiguous:
```
[id=1, name='A', amount=10] [id=2, name='B', amount=20] ...
```
Reading `SUM(amount)` over 1B rows pulls every byte off disk.

**Columnar** — values for one column are contiguous:
```
ids:    [1, 2, 3, ...]
names:  ['A','B','C', ...]
amounts:[10, 20, 30, ...]
```
`SUM(amount)` reads **only the amounts column** — 1/N the I/O.

## Why Columnar Wins for Analytics
| Lever | Effect |
|---|---|
| **Compression** | Same-type values compress 5–20× (RLE, delta, dictionary) |
| **Vectorization** | Process batches in CPU SIMD registers, not row-at-a-time |
| **Late materialization** | Filter/aggregate on encoded columns, decode only survivors |
| **Skip indexes** | Min/max per block → skip whole chunks without reading |

💡 Rule of thumb: if your query touches **>5% of rows but <30% of columns**, columnar is the right tool.

⚠️ Columnar is **terrible** at single-row updates. Don't replace your transactional DB with one — pair them via [[ELT]] / [[CDC]].

## Tags
[[OLAP]] [[OLTP]] [[Columnar]] [[Compression]] [[Vectorization]]
