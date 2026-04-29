# 08 — Deployment

🔑 "Deploying dbt" = scheduling `dbt build` against prod profile + plugging it into CI.

Source: https://docs.getdbt.com/docs/deploy/deployments

## Commands Cheat Sheet
| Command | Does |
|---|---|
| `dbt run` | Builds models only |
| `dbt test` | Runs tests only |
| `dbt build` | run + test + seed + snapshot in DAG order, **fails fast** per node |
| `dbt seed` | Loads `seeds/*.csv` |
| `dbt snapshot` | Runs snapshots |
| `dbt deps` | Installs `packages.yml` |
| `dbt source freshness` | Checks raw landing tables |

💡 In production, prefer `dbt build` — a failing test on a model **stops downstream models from running on bad data**.

## Scheduling — dbt-core
- **cron** — minimal: `0 * * * * cd /repo && dbt build --profiles-dir ~/.dbt`
- **Airflow** — `astronomer-cosmos` parses `manifest.json` into per-model Airflow tasks (granular retries, lineage in UI).
- **Prefect / Dagster** — first-class dbt integrations (`prefect-dbt`, `dagster-dbt`); each model becomes an asset.

## Scheduling — dbt Cloud
- Web UI configures Jobs (cron or webhook).
- Run history, logs, artifacts hosted.
- Built-in CI: PR opens → ephemeral schema rebuilt + tested.

## Selectors
```bash
dbt build -s tag:hourly                  # by tag
dbt build -s marts.finance+              # model + descendants
dbt build -s state:modified+ --defer --state ./prod-manifest
```

## CI with `--defer` + state
🔑 **Slim CI**: only build the **modified** models in your PR; `ref()`s to unmodified upstream resolve to **prod tables** via `--defer`.
1. CI step downloads prod's `manifest.json`.
2. `dbt build -s state:modified+ --defer --state ./prod-state`
3. Saves minutes/dollars per PR.

⚠️ `--defer` requires the prod schema to be readable by the CI service account.

## Tags
[[dbt]] [[Deployment]] [[CI]] [[Airflow]] [[Defer]] [[StateBasedSelection]]
