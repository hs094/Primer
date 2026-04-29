# AnalyticsDB — INDEX

🔑 Two columnar [[OLAP]] engines, opposite deployment models: **[[Clickhouse]]** (server cluster, real-time analytics service) and **[[DuckDB]]** (embedded library, single-node analytics).

## Notes
1. [[Analytics 01 - OLTP vs OLAP]] — row vs columnar, why columnar wins (compression, vectorization, late materialization)
2. [[Analytics 02 - Clickhouse Overview]] — distributed columnar, [[MergeTree]], `ORDER BY` index, TTL, parts/merges
3. [[Analytics 03 - Clickhouse Engines]] — Replacing / Summing / Aggregating / Collapsing MergeTree variants
4. [[Analytics 04 - Clickhouse Distributed]] — sharding (Distributed engine), replication (ReplicatedMergeTree), ClickHouse Keeper
5. [[Analytics 05 - Clickhouse Materialized Views]] — incremental MV pattern, `TO target`, POPULATE, pitfalls
6. [[Analytics 06 - Clickhouse Performance]] — primary key choice, LowCardinality, skip indexes, projections, async inserts
7. [[Analytics 07 - DuckDB Overview]] — embedded "SQLite for OLAP", vectorized, MotherDuck
8. [[Analytics 08 - DuckDB Files and Lake]] — `read_parquet/csv/json`, S3 via httpfs, COPY, ATTACH, Iceberg/Delta/DuckLake
9. [[Analytics 09 - DuckDB in Python]] — `duckdb` package, pandas/polars/arrow interop, Relational API
10. [[Analytics 10 - Comparison]] — ClickHouse vs DuckDB pick guide

## Reference
- ClickHouse docs: https://clickhouse.com/docs
- DuckDB docs: https://duckdb.org/docs/
- MotherDuck: https://motherduck.com/
- DuckLake: https://ducklake.select/

## Related
[[OLAP]] [[Columnar]] [[MergeTree]] [[Parquet]] [[Iceberg]] [[Lakehouse]] [[dbt]]
