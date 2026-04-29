# Observability — INDEX

🔑 Instrument with [[OpenTelemetry]] → ship over [[OTLP]] → store in [[Prometheus]] / [[Loki]] / [[Tempo]] / [[Sentry]] → view in [[Grafana]] → alert on burn rate.

## Reading Order
| # | Note | Topic |
|---|---|---|
| 01 | [[Obs 01 - Three Pillars]] | Logs, metrics, traces (+ profiles, events) |
| 02 | [[Obs 02 - OpenTelemetry Overview]] | API, SDK, OTLP, vendor-neutral |
| 03 | [[Obs 03 - OTel SDK Python]] | `opentelemetry-*` packages, FastAPI auto-instrument |
| 04 | [[Obs 04 - OTel Collector]] | Receivers/processors/exporters pipeline |
| 05 | [[Obs 05 - Semantic Conventions]] | Standard attribute names |
| 06 | [[Obs 06 - Sampling]] | Head vs tail; cost vs fidelity |
| 07 | [[Obs 07 - Prometheus]] | Pull model, exposition, retention |
| 08 | [[Obs 08 - PromQL]] | rate, histogram_quantile, alert idioms |
| 09 | [[Obs 09 - Grafana]] | Dashboards, datasources, alerting |
| 10 | [[Obs 10 - Loki]] | Label-indexed log aggregation, LogQL |
| 11 | [[Obs 11 - Tempo and Distributed Tracing]] | Trace storage + exemplar pivot |
| 12 | [[Obs 12 - Sentry]] | Errors, transactions, releases, profiling |
| 13 | [[Obs 13 - SLOs and Error Budgets]] | SLI/SLO/SLA, multi-burn-rate alerts |

## Mental Model
```
        app code
           │
   OTel API + SDK            ← Obs 02, 03
           │  OTLP            ← Obs 02
           ▼
   OTel Collector             ← Obs 04 (sampling: Obs 06)
       │  │  │
   ┌───┘  │  └────────┐
   ▼      ▼           ▼
 Prom    Loki       Tempo     ← Obs 07, 10, 11
   │      │           │
   └──────┴── Grafana ┘       ← Obs 09
            │
         alerts (burn rate)   ← Obs 13
            │
        Sentry (errors)       ← Obs 12
```

## When To Use What
| Symptom | Reach for |
|---|---|
| "It's slow" | Trace ([[Tempo]]) → profile ([[Sentry]]) |
| "It's broken" | Errors ([[Sentry]]) + logs ([[Loki]]) |
| "Is it healthy?" | Metrics ([[Prometheus]]) + SLO dashboard |
| "Did the deploy break it?" | Release marker on dashboard, Sentry release health |
| "Should I get paged?" | Multi-burn-rate alert (Obs 13) |

## Cross-Cutting Concerns
- **Naming** — always [[Semantic Conventions]] (Obs 05).
- **Cost control** — [[Sampling]] (Obs 06), label cardinality (Obs 07, 10).
- **Correlation** — `trace_id` in every log; `service.name` everywhere.

## Tags
[[Observability]] [[OpenTelemetry]] [[Prometheus]] [[Grafana]] [[Loki]] [[Tempo]] [[Sentry]] [[SLO]]
