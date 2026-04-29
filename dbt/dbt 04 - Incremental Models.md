# 04 — Incremental Models

🔑 Build only the **new/changed rows** since last run. Critical when full rebuild is too slow or expensive.

Source: https://docs.getdbt.com/docs/build/incremental-models

## Anatomy
```sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
) }}

select * from {{ source('events', 'raw') }}
{% if is_incremental() %}
  where event_time > (select max(event_time) from {{ this }})
{% endif %}
```

## `is_incremental()`
True iff: target table exists **and** model is incremental **and** not running `--full-refresh`. Wrap your filter in this guard so first-run / refresh still build everything.

## Strategies
| Strategy | Behavior | Use when |
|---|---|---|
| `append` | Pure `INSERT` | True append-only events, no late updates |
| `merge` | `MERGE` on `unique_key` | Updates allowed — most common |
| `delete+insert` | Delete matching keys, then insert | Warehouses without MERGE |
| `insert_overwrite` | Replace whole partitions | BigQuery / Spark partitioned tables |

## `unique_key`
- String or list of columns.
- With `merge`: matched rows are **updated**, new rows inserted.
- ⚠️ Nulls in the key break matching.

## Late-Arriving Data
🧪 Filter on `updated_at >= dateadd('day', -3, current_date)` to reprocess a rolling window — catches backdated rows.

⚠️ A pure `event_time > max(event_time)` filter **misses late arrivals**. Pick a window wide enough to cover your worst-case lag.

💡 `dbt run --full-refresh -s my_model` rebuilds from scratch when schema changes or logic shifts.

## Tags
[[dbt]] [[Incremental]] [[Merge]] [[CDC]]
