# 09 — DuckDB in Python

🔑 `pip install duckdb`. Query Pandas/Polars/Arrow DataFrames as tables, get results back in any of those formats — zero-copy where possible.

Source: https://duckdb.org/docs/clients/python/overview

## Three Levels of API

### 1. Module-level (one-shot)
```python
import duckdb
duckdb.sql("SELECT 42").show()
```
Implicit global in-memory connection.

### 2. Connection
```python
con = duckdb.connect("warehouse.duckdb")   # or ":memory:"
con.execute("CREATE TABLE t AS SELECT * FROM 'big.parquet'")
```

### 3. Relational API (lazy, composable)
```python
rel = con.sql("SELECT * FROM 'orders.parquet'")
rel.filter("amount > 100").aggregate("country, sum(amount)").show()
```
Each call returns a Relation; nothing executes until you `.fetchall()` / `.df()` / `.show()`.

## DataFrame Interop
🔑 DuckDB scans your in-process DataFrame **by reference** — no copy, no import.
```python
import pandas as pd
df = pd.read_csv("orders.csv")          # pandas
duckdb.sql("SELECT country, sum(amount) FROM df GROUP BY 1").show()

import polars as pl
pdf = pl.read_parquet("orders.parquet") # polars
duckdb.sql("SELECT * FROM pdf WHERE amount > 100").pl()  # back to polars
```

## Result Conversion
| Method | Returns |
|---|---|
| `.fetchall()` | list[tuple] |
| `.df()` | pandas DataFrame |
| `.pl()` | polars DataFrame |
| `.arrow()` | pyarrow Table |
| `.fetchnumpy()` | dict[str, np.ndarray] |

## Replacing pandas for Medium Data
🧪 `df.groupby('country')['amount'].sum()` works fine at 1M rows, dies at 100M.
Same query in DuckDB:
```python
duckdb.sql("SELECT country, sum(amount) FROM df GROUP BY country").df()
```
- Streams instead of loading whole intermediate columns.
- Multithreaded out-of-the-box.
- Spills to disk past RAM.

⚠️ DataFrame access is **read-only** from DuckDB's side — `INSERT INTO df` won't work; create a DuckDB table first.

💡 In notebooks, treat DuckDB as your "SQL kernel" — hand off pandas DataFrames in, get DataFrames out.

## Tags
[[DuckDB]] [[Python]] [[Pandas]] [[Polars]] [[Arrow]]
