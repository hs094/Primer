# 08 — DuckDB: Files and Lakehouse

🔑 DuckDB treats files (Parquet, CSV, JSON) and remote object stores (S3, GCS, HTTP) as **tables**. No ingest step required.

Source: https://duckdb.org/docs/data/parquet/overview

## Reading Files
```sql
-- single file (extension auto-detected)
SELECT * FROM 'orders.parquet';

-- glob across many
SELECT * FROM read_parquet('s3://bucket/orders/year=*/month=*/*.parquet');

-- CSV with auto-detect
SELECT * FROM read_csv_auto('events.csv');

-- newline JSON
SELECT * FROM read_json_auto('logs.ndjson');

-- list of files
SELECT * FROM read_parquet(['a.parquet', 'b.parquet']);
```
Parquet/CSV/JSON readers support **predicate + projection pushdown** — only the columns/row groups you touch get fetched.

## Writing Files (`COPY`)
```sql
COPY (SELECT * FROM big_table WHERE day = '2024-01-01')
TO 's3://bucket/exports/day=2024-01-01.parquet'
(FORMAT parquet, COMPRESSION zstd);

-- Hive-style partitioning
COPY events TO 's3://bucket/events'
(FORMAT parquet, PARTITION_BY (year, month), OVERWRITE_OR_IGNORE);
```

## Remote Storage — `httpfs`
```sql
INSTALL httpfs; LOAD httpfs;
SET s3_region='us-east-1';
SET s3_access_key_id='...'; SET s3_secret_access_key='...';
SELECT count(*) FROM 's3://bucket/data.parquet';
```
Also supports GCS (`gs://`) and any HTTPS URL.

## `ATTACH` — Cross-database
```sql
-- attach another DuckDB file
ATTACH 'staging.duckdb' AS stg;
SELECT * FROM stg.main.orders;

-- attach Postgres / SQLite / MySQL
ATTACH 'host=db port=5432 dbname=app user=ro' AS pg (TYPE postgres);
```

## Lakehouse Formats
- **Iceberg**: `INSTALL iceberg; LOAD iceberg; SELECT * FROM iceberg_scan('s3://.../table');`
- **Delta**: `INSTALL delta; LOAD delta; SELECT * FROM delta_scan('s3://.../delta_table');`
- **DuckLake** (2024) — DuckDB's own table format: ACID multi-writer over Parquet + a SQL catalog DB. Open-source alternative to Iceberg.

💡 The whole "lakehouse on a laptop" pattern: DuckDB + S3 + Parquet/Iceberg = no warehouse needed.

## Tags
[[DuckDB]] [[Parquet]] [[S3]] [[Iceberg]] [[DuckLake]] [[Lakehouse]]
