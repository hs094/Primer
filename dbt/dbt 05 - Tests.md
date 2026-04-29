# 05 — Tests

🔑 A dbt test is a `SELECT` that returns **failing rows**. Zero rows = pass.

Source: https://docs.getdbt.com/docs/build/data-tests

## Generic Tests (the four built-ins)
Declared in `models/schema.yml`:
```yaml
models:
  - name: stg_orders
    columns:
      - name: order_id
        tests: [unique, not_null]
      - name: status
        tests:
          - accepted_values: { values: ['placed','shipped','cancelled'] }
      - name: customer_id
        tests:
          - relationships: { to: ref('stg_customers'), field: customer_id }
```

| Test | Asserts |
|---|---|
| `unique` | No duplicate values |
| `not_null` | No NULLs |
| `accepted_values` | Column ⊆ allowed set |
| `relationships` | FK exists in parent |

## Singular Tests
One-off `.sql` in `tests/` returning failing rows:
```sql
-- tests/no_negative_amounts.sql
select * from {{ ref('fct_payments') }} where amount < 0
```

## Packages: dbt-expectations
Expectations-style suite (à la Great Expectations):
```yaml
- dbt_expectations.expect_column_values_to_be_between:
    min_value: 0
    max_value: 1000
- dbt_expectations.expect_row_values_to_have_recent_data:
    datepart: day
    interval: 1
```

## Severity & Storage
```yaml
tests:
  - unique:
      config:
        severity: warn        # or 'error'
        warn_if: ">10"
        error_if: ">100"
        store_failures: true  # writes failing rows to dbt_test__audit
```

💡 `store_failures: true` → debug failing rows by querying the audit schema instead of re-running.

⚠️ `dbt test` returns nonzero exit code on `error` severity → use this to fail CI.

## Tags
[[dbt]] [[Testing]] [[DataQuality]] [[dbt-expectations]]
