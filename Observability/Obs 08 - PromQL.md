# 08 — PromQL

🔑 Pick a vector type, apply a function, aggregate. Most alert queries are `rate()` → `sum by (...)` → comparison.

Source: https://prometheus.io/docs/prometheus/latest/querying/basics/

## Vector Types
| Type | Example | Use |
|---|---|---|
| **Instant** | `http_requests_total` | "value right now" |
| **Range** | `http_requests_total[5m]` | input to rate-style functions |

## The Big Functions
```promql
# Per-second rate, averaged over 5m (use on counters!)
rate(http_requests_total[5m])

# Most-recent-two-points slope (spiky, dashboards only — never alerts)
irate(http_requests_total[5m])

# 95th percentile latency from histogram buckets
histogram_quantile(0.95, sum by (le) (rate(request_duration_seconds_bucket[5m])))

# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
```

## Aggregation
```promql
# Group: collapse all labels except 'route'
sum by (route)  (rate(http_requests_total[5m]))

# Drop only one label
sum without (instance) (rate(http_requests_total[5m]))
```
Operators: `sum`, `avg`, `min`, `max`, `count`, `topk`, `quantile`.

## Alert Idioms
```promql
# Error rate > 5%
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) > 0.05

# p99 latency > 500ms
histogram_quantile(0.99,
  sum by (le, service) (rate(request_duration_seconds_bucket[5m]))
) > 0.5

# A pod has been crashlooping for 10m
increase(kube_pod_container_status_restarts_total[10m]) > 3
```

## Gotchas
- ⚠️ `rate` on a **gauge** is nonsense — use only on counters.
- ⚠️ `rate(x[1m])` with 15s scrape = 4 samples. Window must cover ≥ 4× scrape interval.
- 💡 Always pre-aggregate (`sum by ...`) *before* `histogram_quantile` — quantile of an average is wrong.

## Tags
[[Prometheus]] [[PromQL]] [[Metrics]] [[SLO]]
