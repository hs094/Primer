# 07 — Prometheus

🔑 Pull-based, label-indexed time-series DB. Scrapes `/metrics` over HTTP, stores locally, queries via PromQL.

Source: https://prometheus.io/docs/introduction/overview/

## Model
- **Pull, not push.** Prometheus scrapes targets on a fixed `scrape_interval` (default 15s).
- Time series = `metric_name{label1="v1", label2="v2"}` → stream of `(timestamp, float64)`.
- 4 metric types: **counter**, **gauge**, **histogram**, **summary**.

## Exposition Format (`/metrics`)
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="POST",route="/login",status="200"} 1027
http_requests_total{method="POST",route="/login",status="500"} 3
# TYPE request_duration_seconds histogram
request_duration_seconds_bucket{le="0.1"} 24054
request_duration_seconds_bucket{le="+Inf"} 24812
request_duration_seconds_sum   8953.3
request_duration_seconds_count 24812
```

## Scrape Config
```yaml
scrape_configs:
  - job_name: api
    scrape_interval: 15s
    static_configs:
      - targets: ['api:8000']
  - job_name: k8s-pods
    kubernetes_sd_configs: [{ role: pod }]
```

## Retention & Long-Term Storage
- Local TSDB: default **15 days** (`--storage.tsdb.retention.time=15d`).
- Long-term: `remote_write` to Mimir / Cortex / Thanos / VictoriaMetrics; `remote_read` for queries.
- ⚠️ Local disk is fine for one box; for HA / multi-month data, always remote_write.

## Push Gateway
- For **short-lived batch jobs** that exit before scrape. Don't use for services.

## Gotchas
- ⚠️ "Not 100% accurate" — fine for monitoring, never for billing.
- ⚠️ Cardinality kills: each unique label combo = one series. Avoid `user_id`, `trace_id` as labels.
- 💡 Use [[Semantic Conventions]] names so Grafana dashboards port across services.

## Tags
[[Prometheus]] [[PromQL]] [[Metrics]] [[Grafana]]
