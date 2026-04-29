# 06 — Macros and Jinja

🔑 Jinja turns dbt SQL into a **programming environment** — control flow, variables, functions — that compiles down to plain SQL.

Source: https://docs.getdbt.com/docs/build/jinja-macros

## Jinja Syntax
| Delim | Use |
|---|---|
| `{{ ... }}` | Expression — outputs a value |
| `{% ... %}` | Statement — control flow, no output |
| `{# ... #}` | Comment |

```sql
{% set payment_methods = ['credit', 'ach', 'wire'] %}
select order_id,
{% for m in payment_methods %}
  sum(case when method = '{{ m }}' then amount end) as {{ m }}_total
  {%- if not loop.last %},{% endif %}
{% endfor %}
from {{ ref('stg_payments') }} group by 1
```

## Macros = Reusable SQL Functions
```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(col, scale=2) %}
  ({{ col }} / 100)::numeric(16, {{ scale }})
{% endmacro %}
```
Call: `select {{ cents_to_dollars('amount_cents') }} as amount`

## `{{ config() }}`
Per-model override of materialization, tags, hooks, etc.:
```sql
{{ config(materialized='incremental', tags=['hourly']) }}
```

## Packages — `dbt_utils`
`packages.yml`:
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
  - package: calogica/dbt_expectations
    version: 0.10.0
```
Then `dbt deps`. Useful macros: `dbt_utils.surrogate_key`, `pivot`, `star`, `date_spine`, `generate_series`.

💡 Check **dbt-utils first** before writing a macro — most "I need a helper for X" already exists.

⚠️ Over-engineered Jinja kills readability. DRY ≠ unintelligible.

## Tags
[[dbt]] [[Jinja]] [[Macros]] [[dbt_utils]] [[Templating]]
