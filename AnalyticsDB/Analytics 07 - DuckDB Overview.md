# 07 — DuckDB Overview

🔑 [[DuckDB]] is the **"SQLite for OLAP"** — an embedded, in-process columnar database. No server, no daemon, no deploy. Just a library you link into your app.

Source: https://duckdb.org/docs/

## Identity
- Embedded — runs **inside** your Python/R/Java/JS process.
- Columnar storage + **vectorized execution** engine (batches of ~2048 values through CPU SIMD).
- ACID on a single `.duckdb` file (or pure in-memory).
- Pure C++ core, MIT license, single binary, zero dependencies.

## What You Get
| Feature | Meaning |
|---|---|
| Native Parquet/CSV/JSON | Query files directly with no import: `SELECT * FROM 'data.parquet'` |
| Pandas / Polars / Arrow zero-copy | DataFrames are first-class tables |
| Full SQL | Window functions, CTEs, recursive, JSON ops, GIS via extension |
| Extensions | `httpfs`, `iceberg`, `delta`, `spatial`, `fts`, `vss` (vectors) |

## Hello World
```python
import duckdb
duckdb.sql("SELECT count(*) FROM 'orders/*.parquet'").show()
```
Or persistent:
```python
con = duckdb.connect("warehouse.duckdb")
con.execute("CREATE TABLE t AS SELECT * FROM read_csv_auto('big.csv')")
```

## When DuckDB Wins
- **Notebook / ad-hoc analytics** — laptop-scale data (up to 100s of GB).
- **Single-node ETL** — replace a pandas / Spark job with SQL on Parquet.
- **dbt-duckdb** — full [[dbt]] stack against local files / S3, no warehouse bill.
- **Lakehouse query engine** — read Iceberg/Delta from object storage.

## MotherDuck
🔑 Cloud DuckDB — same engine, **hybrid execution**: queries can straddle local + cloud data, you pick where each table lives. Adds collaboration, persistence, and bigger compute.

⚠️ Single-writer. For concurrent OLAP serving over the network, use [[Clickhouse]].

💡 Heuristic: if it fits on one beefy box and you don't need 24/7 service, DuckDB.

## Tags
[[DuckDB]] [[Embedded]] [[Vectorized]] [[Columnar]] [[Parquet]] [[MotherDuck]]
