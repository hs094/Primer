# 10 — ClickHouse vs DuckDB: Pick Guide

🔑 Both are columnar OLAP. They live at **opposite ends of the deployment spectrum**: server cluster vs in-process library. Pick by where the query runs, not by SQL features.

## Side-by-Side
| Dim | [[Clickhouse]] | [[DuckDB]] |
|---|---|---|
| Deployment | Server / cluster (or Cloud) | Embedded library, in-process |
| Storage | Own format (parts) on disk/S3 | `.duckdb` file or remote files |
| Concurrency | Many concurrent writers + readers | Single writer, many readers |
| Scale | Petabyte, multi-node sharded | Single node, ~TB practical ceiling |
| Best workload | Real-time dashboards, high-QPS analytics | Ad-hoc / notebook / single-node ETL |
| Latency | sub-second on billions | sub-second on 100s of GB |
| Ingest | Streaming, Kafka, async inserts | Batch — `INSERT`, `COPY`, `read_*` |
| Replication | ReplicatedMergeTree + Keeper | None (file-level backup) |
| Materialized views | Insert-trigger MVs (Note 05) | Standard SQL views, no MV |
| Ecosystem | Grafana, Superset, Metabase | Pandas, Polars, Arrow, dbt-duckdb |
| Licence | Apache 2.0 | MIT |

## When to Pick Which
**Reach for [[Clickhouse]] when:**
- You need a 24/7 service answering many users' queries concurrently.
- Data exceeds a single beefy box, or you ingest >100k events/sec.
- You're building a product feature (real-time analytics dashboard, observability, ad-tech).
- You need replication / HA / multi-tenant ACL.

**Reach for [[DuckDB]] when:**
- One analyst, one notebook — you want SQL over Parquet/CSV with zero ops.
- You're replacing a pandas/Spark batch job at single-node scale.
- You're running [[dbt]] over files on S3 with no warehouse bill.
- You need to query a lake (Iceberg/Delta) without spinning up Trino.

## Combined Pattern
🧪 The two compose:
1. Production app → ClickHouse (live OLAP).
2. Analyst → DuckDB on a Parquet export of the same data, in a notebook.
Both speak (mostly) the same SQL.

⚠️ Neither replaces a transactional DB. Front them with [[PostgreSQL]] for OLTP.

💡 Heuristic: if the question is "should this run on a server?", pick ClickHouse. If "should this run in my Python process?", pick DuckDB.

## Tags
[[Clickhouse]] [[DuckDB]] [[OLAP]] [[Comparison]] [[Lakehouse]]
