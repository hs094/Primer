# 01 — Overview

🔑 [[dbt]] is the **T in [[ELT]]** — a SQL transformation framework that compiles your `SELECT` statements into tables/views inside your warehouse.

Source: https://docs.getdbt.com/docs/introduction

## Mental Model
- Raw data is already loaded into your warehouse (Snowflake, BigQuery, Redshift, [[Clickhouse]], [[DuckDB]], Postgres).
- You write **models** = `SELECT` statements in `.sql` files.
- dbt compiles them into `CREATE TABLE` / `CREATE VIEW` DDL and runs it in-warehouse.
- No external compute — the warehouse does the heavy lifting.

## Why It Matters
| Concern | Pre-dbt | dbt |
|---|---|---|
| Versioning | SQL in BI tool / scripts | Git-tracked models |
| Testing | Manual / none | `dbt test` against assertions |
| Lineage | Tribal knowledge | Auto DAG from `{{ ref() }}` |
| Docs | Wiki rot | Generated from YAML + SQL |
| Reuse | Copy-paste | Macros + packages |

## dbt-Core vs dbt Cloud
- **dbt-core** — OSS Python CLI. You run `dbt run`, `dbt test`, `dbt build`. Schedule with cron / [[Airflow]] / Prefect / Dagster.
- **dbt Cloud (dbt Platform)** — managed: web IDE, scheduler, CI/CD, lineage UI, semantic layer, alerting.

💡 Analytics engineers ship production-grade pipelines with **just SQL + Jinja** — no Python or Spark required.

⚠️ dbt does *not* extract or load. Pair with [[Fivetran]] / [[Airbyte]] / custom EL for the EL.

## Tags
[[dbt]] [[ELT]] [[SQL]] [[DataWarehouse]] [[AnalyticsEngineering]]
