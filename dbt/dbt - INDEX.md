# dbt — INDEX

🔑 [[dbt]] is the **T in [[ELT]]**: a SQL transformation framework that compiles models into your warehouse.

## Notes
1. [[dbt 01 - Overview]] — what dbt is, dbt-core vs Cloud, mental model
2. [[dbt 02 - Project Structure]] — `dbt_project.yml`, `profiles.yml`, directory layout
3. [[dbt 03 - Models and Materializations]] — view / table / incremental / ephemeral / mat-view, `ref()`, `source()`, DAG
4. [[dbt 04 - Incremental Models]] — `is_incremental()`, `unique_key`, merge / append / delete+insert, late data
5. [[dbt 05 - Tests]] — generic + singular tests, dbt-expectations, severity, `store_failures`
6. [[dbt 06 - Macros and Jinja]] — Jinja syntax, macros, `config()`, `dbt_utils`, packages
7. [[dbt 07 - Snapshots and SCD2]] — `dbt_valid_from/to`, timestamp vs check strategy
8. [[dbt 08 - Deployment]] — `dbt build`, cron / Airflow / Prefect / Dagster, dbt Cloud, `--defer` slim CI

## Reference
- Docs: https://docs.getdbt.com/
- Core repo: https://github.com/dbt-labs/dbt-core
- Discourse: https://discourse.getdbt.com/

## Related
[[ELT]] [[SCD2]] [[DataWarehouse]] [[AnalyticsEngineering]] [[Airflow]]
