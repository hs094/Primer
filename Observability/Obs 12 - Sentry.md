# 12 — Sentry

🔑 Errors-first APM: stack traces with local variables + transactions = traces, releases, profiling. One SDK, two integrations, done.

Source: https://docs.sentry.io/platforms/python/

## What You Get
| Feature | Notes |
|---|---|
| **Error capture** | Full stack with frame-local vars + breadcrumbs |
| **Transactions** | Sentry's name for traces; works alongside / instead of OTel |
| **Releases** | Tag events with `release=app@1.4.2`; regression detection |
| **Source maps** | JS/TS only — minified frames → original source |
| **Performance** | p50/p95 per route, slow-query / slow-HTTP detection |
| **Profiling** | Sampled CPU profiles on slow transactions |

## Python Setup (FastAPI + asyncio)
```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

sentry_sdk.init(
    dsn=settings.sentry_dsn,
    environment=settings.env,                # "prod" | "staging"
    release=f"api@{settings.version}",
    traces_sample_rate=0.1,                  # 10% of transactions
    profiles_sample_rate=1.0,                # of *those* transactions
    send_default_pii=False,                  # flip True only if you've audited
    integrations=[FastApiIntegration(), SqlalchemyIntegration()],
)
```

## Capture Patterns
```python
# Manual
try:
    await charge(user)
except StripeError as e:
    sentry_sdk.capture_exception(e)
    raise

# Custom span inside a transaction
with sentry_sdk.start_span(op="db.query", name="fetch_user"):
    user = await repo.get(user_id)

# Tag + context
sentry_sdk.set_tag("tenant", tenant_id)
sentry_sdk.set_user({"id": user_id})
```

## Sentry vs OTel
- Not exclusive. OTel SDK can ship to Sentry via OTLP (Sentry has an OTLP receiver in beta), or run Sentry SDK alongside OTel.
- 💡 Use Sentry for **errors + release health**, OTel/Tempo for **deep tracing**.

## Gotchas
- ⚠️ `send_default_pii=True` ships request bodies + headers. Audit before prod.
- ⚠️ `traces_sample_rate=1.0` in prod = $$$. Start at 0.05–0.1, raise on staging.

## Tags
[[Sentry]] [[Errors]] [[Python]] [[FastAPI]] [[Releases]]
