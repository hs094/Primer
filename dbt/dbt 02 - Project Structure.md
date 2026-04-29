# 02 — Project Structure

🔑 A dbt project is just a directory tree with one required config file (`dbt_project.yml`) and conventions for where each artifact type lives.

Source: https://docs.getdbt.com/docs/build/projects

## Layout
```
my_project/
├── dbt_project.yml      # project config (required)
├── packages.yml         # external packages (dbt_utils, etc.)
├── models/              # SELECT statements -> tables/views
│   ├── staging/         # 1:1 cleanups of sources
│   ├── intermediate/    # joins, business logic
│   └── marts/           # final consumer tables
├── seeds/               # static CSVs loaded as tables
├── snapshots/           # SCD2 history (see Note 07)
├── tests/               # singular SQL tests
├── macros/              # reusable Jinja macros
├── analyses/            # ad-hoc SQL not materialized
└── target/              # compiled SQL + run artifacts (gitignored)
```

## `dbt_project.yml` — the manifest
```yaml
name: 'analytics'
version: '1.0.0'
profile: 'warehouse'      # -> picks connection from profiles.yml
model-paths: ["models"]
models:
  analytics:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

## `profiles.yml` — credentials
🔑 Lives **outside** the project (default `~/.dbt/profiles.yml`) so secrets don't get committed.
```yaml
warehouse:
  target: dev
  outputs:
    dev: { type: postgres, host: ..., user: ..., password: ..., schema: dbt_dev }
    prod: { type: postgres, host: ..., schema: analytics }
```

⚠️ The `profile:` key in `dbt_project.yml` must match the top-level key in `profiles.yml`.

💡 Convention: `staging.*` = view, `marts.*` = table. Tune per layer in `dbt_project.yml`.

## Tags
[[dbt]] [[ProjectStructure]] [[YAML]] [[Profiles]]
