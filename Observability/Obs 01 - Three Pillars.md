# 01 — Three Pillars

🔑 Logs, metrics, traces — same telemetry seen at different angles. Profiles + events are emerging fourth/fifth pillars.

Source: https://opentelemetry.io/docs/concepts/observability-primer/

## The Three (Plus Two)
| Pillar | What it is | Best for |
|---|---|---|
| **Logs** | Timestamped messages from services | Debugging specific events; carries free-form context |
| **Metrics** | Numeric aggregates over time | SLIs/SLOs, dashboards, trend + capacity |
| **Traces** | Path of one request across services | Distributed root-cause, latency breakdown |
| **Profiles** (emerging) | CPU/heap samples per function | "Why is this span slow inside the process?" |
| **Events** (emerging) | Discrete, structured business signals | Releases, deploys, feature-flag flips overlaid on time series |

## When To Reach For Which
- Service is **slow**: trace first, then profile.
- Service is **erroring**: log + trace (correlate via trace_id).
- Service is **trending bad**: metric (rate, latency p95, error %).
- "Did the deploy cause it?": event marker on dashboard.

## Correlation Beats Pillar Choice
- 💡 The win is *correlation*: every log/metric carries `trace_id`, `span_id`, and a stable `service.name` (see [[Semantic Conventions]]).
- ⚠️ Logs without trace context = expensive grep. Always inject trace_id into log records (OTel logging handler does this for you).

## Cost Profile (rough)
- Metrics: cheapest (pre-aggregated, low cardinality).
- Logs: medium-high (volume scales with traffic).
- Traces: high without sampling — see [[Sampling]].

## Tags
[[OpenTelemetry]] [[Logs]] [[Metrics]] [[Traces]] [[Profiles]]
