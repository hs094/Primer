# 10 — Loki

🔑 "Prometheus for logs" — index *labels only*, store compressed log chunks in object storage. Cheap, label-driven.

Source: https://grafana.com/docs/loki/latest/

## Why Different From Elasticsearch
| | Loki | Elasticsearch |
|---|---|---|
| Index | Labels (small) | Full text (huge) |
| Storage | Object store (S3/GCS) | Local SSD shards |
| Query | LogQL (Prom-like) | Lucene/KQL |
| Cost | $ | $$$ |
| Trade-off | Slow on adhoc full-text grep | Fast everything, expensive |

## Push, Don't Pull
- Loki is **push-based** (HTTP `/loki/api/v1/push`).
- Shippers: **Alloy** (current), **Promtail** (legacy), Fluent Bit, Vector, Docker driver, OTLP logs.

## LogQL Cheat Sheet
```logql
# Stream selector (labels REQUIRED)
{service="api", env="prod"}

# Filter on content
{service="api"} |= "error" != "healthcheck"

# Parse JSON, filter, count
sum by (status) (
  rate({service="api"} | json | status=~"5.." [5m])
)

# Convert log line to metric (alertable)
sum(rate({service="api"} |= "OOM" [5m]))
```

## Label Hygiene
- ⚠️ **Labels = index keys.** High-cardinality labels (user_id, trace_id, request_id) destroy Loki. Put those in the log *content*, not labels.
- Good labels: `service`, `env`, `level`, `cluster`, `namespace`. ~10 max per stream.

## Trace Correlation
- Inject `trace_id` into log lines (OTel logging handler does this).
- In Grafana, derived field `trace_id` → click to [[Tempo]].

## Tags
[[Loki]] [[Logs]] [[LogQL]] [[Grafana]]
