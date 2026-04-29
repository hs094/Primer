# 07 — Snapshots and SCD2

🔑 A **snapshot** captures every change to a mutable source row over time — implementing [[SCD2]] (Slowly Changing Dimension Type 2).

Source: https://docs.getdbt.com/docs/build/snapshots

## The Problem
Source `orders.status` is overwritten in-place: `pending` → `shipped`. You lose history. Snapshots solve this by inserting a new row each time a tracked field changes.

## Schema Additions
| Column | Meaning |
|---|---|
| `dbt_valid_from` | When this version of the row started |
| `dbt_valid_to` | When it ended (NULL for current row) |
| `dbt_scd_id` | Surrogate key per snapshot row |
| `dbt_updated_at` | Mirrors source's update timestamp |

## Two Strategies
### Timestamp (preferred)
Source has a reliable `updated_at` column:
```sql
{% snapshot orders_snapshot %}
{{ config(
    target_schema='snapshots',
    unique_key='order_id',
    strategy='timestamp',
    updated_at='updated_at'
) }}
select * from {{ source('jaffle', 'orders') }}
{% endsnapshot %}
```

### Check
No reliable timestamp — diff specific columns:
```sql
{{ config(
    strategy='check',
    unique_key='order_id',
    check_cols=['status', 'amount']
) }}
```

## Querying SCD2
"Status as of 2024-06-01":
```sql
select * from {{ ref('orders_snapshot') }}
where '2024-06-01' >= dbt_valid_from
  and ('2024-06-01' <  dbt_valid_to or dbt_valid_to is null)
```

💡 Run snapshots **frequently** (hourly) — you only capture what's there *at run time*. Changes between runs collapse.

⚠️ Once snapshotted, never `--full-refresh` a snapshot — you destroy history.

## Tags
[[dbt]] [[SCD2]] [[Snapshot]] [[History]] [[DimensionalModel]]
