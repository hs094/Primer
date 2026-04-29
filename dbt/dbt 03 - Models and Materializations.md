# 03 — Models and Materializations

🔑 A **model** is a `SELECT` statement. The **materialization** decides what DDL dbt emits to persist it.

Source: https://docs.getdbt.com/docs/build/materializations

## The Five Built-ins
| Materialization | DDL | When to use |
|---|---|---|
| `view` | `CREATE VIEW` | Cheap, always-fresh, light transforms |
| `table` | `CREATE TABLE AS` | Expensive query, fast reads |
| `incremental` | Insert/merge new rows | Append-only event data, big tables |
| `ephemeral` | Inlined as CTE in dependents | Internal helper, never materialized |
| `materialized_view` | `CREATE MATERIALIZED VIEW` | Auto-refresh by warehouse (BQ, Snowflake, [[Clickhouse]]) |

## Configuring
Inline in the model:
```sql
{{ config(materialized='table') }}
select user_id, count(*) as orders
from {{ ref('stg_orders') }}
group by 1
```

Or in `dbt_project.yml` for a whole folder.

## `ref()` and `source()` — the DAG glue
🔑 Never hardcode table names. Use:
- `{{ ref('stg_orders') }}` — refers to another model. dbt builds dependency graph.
- `{{ source('jaffle', 'raw_orders') }}` — refers to a raw landed table declared in `sources.yml`.

```sql
-- models/marts/fct_orders.sql
select o.*, c.name
from {{ ref('stg_orders') }} o
left join {{ ref('stg_customers') }} c using (customer_id)
```

## DAG Resolution
1. dbt parses every model's `ref()`/`source()` calls.
2. Builds a DAG.
3. `dbt run` topologically sorts and executes — parallelism per `--threads`.

💡 `dbt compile` shows you the rendered SQL with table names substituted.

⚠️ Circular `ref()`s are caught at parse time and abort the run.

## Tags
[[dbt]] [[Materialization]] [[DAG]] [[ref]] [[source]]
