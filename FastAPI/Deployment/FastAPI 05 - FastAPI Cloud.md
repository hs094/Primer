# Deploy 05 — FastAPI Cloud

🔑 FastAPI Cloud is the official managed hosting platform from the FastAPI team — opinionated PaaS designed around the framework.

## What it offers

- Push-to-deploy from a Git repo / CLI.
- HTTPS, custom domains, autoscaling Uvicorn workers.
- Logs, metrics, secret management built-in.
- Aimed at "deploy a FastAPI app without owning infra".

## CLI flow

`fastapi[standard]` already pulls in `fastapi-cli[standard]`, which pulls `fastapi-cloud-cli` and registers `deploy` / `login` directly on the `fastapi` command:

```bash
pip install "fastapi[standard]"       # or: pip install fastapi-cloud-cli
fastapi login
fastapi deploy
```

More subcommands live under `fastapi cloud …` (e.g. `fastapi cloud logs`, `fastapi cloud whoami`, `fastapi cloud env`). The CLI bundles your app, uploads, and runs it behind their load balancer.

## When to use it

- You want zero-ops.
- Your app fits the "FastAPI + Postgres + maybe Redis" mould.
- You're early-stage / prototyping and don't need fine-grained infra control.

## When *not* to use it

- You already run k8s / ECS — your platform team has CI/CD opinions.
- You need provider-specific services (BigQuery, S3 lifecycle, etc.) co-located.
- Compliance requires a specific cloud / region.

## Local-first alternatives

For self-managed deploys:
- [[FastAPI 03 - Docker]] — image you can run anywhere.
- [[FastAPI 06 - Run a Server Manually]] — Uvicorn / Gunicorn details.
- [[FastAPI 07 - Cloud Providers]] — AWS/GCP/Azure patterns.

## Cost vs DIY

Managed = predictable infra cost + saved engineering hours. DIY = lower marginal cost but ops overhead. Lean managed early; revisit when scale forces it.

💡 Don't pick the platform first — write the app, then deploy whichever path matches your team's operational appetite.
