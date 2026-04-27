# TC 10 — Cloud Metrics

🔑 **Scrape `metrics.temporal.io` to see what Cloud sees about your account — request rates, errors, throttling, schedule health — and pair it with SDK metrics for the full picture.**

## Two metric sources, two perspectives
Temporal Cloud emits two complementary streams:

- **Cloud Metrics** — what the Temporal Service sees: requests, errors, resource exhaustion, schedule actions, replication lag. Tells you about the **platform side** of every gRPC call.
- **SDK Metrics** — what your [[Temporal EN 06 - Workers]] see: task latency, poll success, sticky cache hit rate, Workflow/Activity outcomes. Tells you about the **client side**. See [[Temporal RF 01 - SDK Metrics]] and [[Temporal SH 08 - Monitoring]].

You need both. Cloud metrics answer "is Temporal healthy for me?"; SDK metrics answer "are my workers keeping up?". For wider monitoring strategy see [[Temporal SH 08 - Monitoring]].

## OpenMetrics endpoint
Cloud exposes a **Prometheus-compatible scrapable endpoint at `metrics.temporal.io`**. Authentication is done with an API key tied to a Service Account holding the **Metrics Read-Only** role (see [[Temporal TC 08 - Users and Groups]] and [[Temporal TC 11 - Billing and Usage]]).

Series live under the `temporal_cloud_v1_*` namespace. Scrape from Prometheus, Grafana Cloud, Datadog, New Relic, or anything that speaks OpenMetrics.

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: temporal-cloud
    scheme: https
    metrics_path: /prometheus
    bearer_token: ${TEMPORAL_CLOUD_API_KEY}
    static_configs:
      - targets: ["metrics.temporal.io"]
```

## What you'll typically chart
Common series (browse the full catalog at the docs):

- `temporal_cloud_v1_frontend_service_request_count` — RPS by namespace and operation.
- `temporal_cloud_v1_frontend_service_error_count` — error counts; pair with the success ratio.
- `temporal_cloud_v1_resource_exhausted_error_count` — your throttling signal. Non-zero means Cloud is rate-limiting your account.
- `temporal_cloud_v1_schedule_action_success_count` — Schedules executing as configured.
- Replication-lag gauges for [[Temporal TC 13 - High Availability]] namespaces.

## Useful PromQL
```promql
# Frontend error ratio per namespace, 5m window
sum by (namespace) (
  rate(temporal_cloud_v1_frontend_service_error_count[5m])
)
/
sum by (namespace) (
  rate(temporal_cloud_v1_frontend_service_request_count[5m])
)

# Throttling alert
sum by (namespace) (
  rate(temporal_cloud_v1_resource_exhausted_error_count[5m])
) > 0
```

A non-zero `resource_exhausted` rate means you're hitting per-namespace RPS limits — usually it's pollers, not workflow logic. Tune Worker poller counts before requesting a limit increase.

## Deprecated PromQL endpoint
The legacy mTLS-based PromQL endpoint was **deprecated on April 2, 2026** and stops accepting new users; full discontinuation is **October 5, 2026**. Migrate to the OpenMetrics endpoint with API-key auth.

⚠️ The mTLS metrics path is separate from your namespace mTLS — different cert, different rotation cadence. Don't conflate them with [[Temporal TC 12 - Connectivity]] data-plane certs.

## Dashboards
Temporal publishes example Grafana dashboards covering frontend traffic, error rate, latency percentiles, schedules, and replication. Import them, then layer SDK metrics on the same boards so a single alert path covers both sides. See [[Temporal RF 06 - Service Configuration]] for the dial-tone metrics to watch in production.

## Alerts worth wiring
- Sustained `resource_exhausted` rate > 0 — request a limit bump or reduce pollers.
- Frontend error ratio > 1% over 5m — open Temporal Support, check status page.
- Schedule action failure rate > 0 — broken Schedules; pair with Workflow failure metrics.
- Replication lag above your RPO budget — see [[Temporal TC 13 - High Availability]].

## Migration from v0 to v1
The legacy series carried different names and labels. The v1 series under `temporal_cloud_v1_*` is the canonical set going forward:

- New names use the `temporal_cloud_v1_` prefix consistently.
- Label cardinality is tighter — fewer one-off labels per series.
- Keep both endpoints scraping in parallel through your migration window, then drop v0 dashboards once v1 alerts have soaked.

If you have existing dashboards built against v0, update them and the alert rules together — series names changed, so silently broken alerts are the failure mode.

## Integrations
The OpenMetrics endpoint plugs into anything that scrapes Prometheus:

- **Datadog** — OpenMetrics check or Agent integration; tag by `namespace`.
- **Grafana Cloud** — direct scrape with the API key; import the published dashboards.
- **New Relic** — Prometheus remote-write or OpenMetrics integration.
- **Self-hosted Prometheus + Grafana** — straight scrape config above; pair with [[Temporal RF 06 - Service Configuration]] for what to alert on.

For a self-hosted comparison see [[Temporal SH 08 - Monitoring]] — the dashboards translate, but the metric prefix differs from open-source `temporal_*`.

## Access control
Granting a Service Account the **Metrics Read-Only** role is enough to scrape — no namespace permissions required. Rotate the key on the same cadence as your other [[Temporal TC 04 - API Keys]] and put alerts on the key expiry notifications from [[Temporal TC 14 - Cloud Operations and tcld]].

💡 **Takeaway:** Cloud Metrics are your view into Temporal's behavior on your behalf — scrape them, alert on `resource_exhausted` and error ratio, and combine with SDK metrics to triage from request-in to task-done. Migrate off the mTLS PromQL endpoint before October 2026.
