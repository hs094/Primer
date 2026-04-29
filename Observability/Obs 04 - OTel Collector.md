# 04 — OTel Collector

🔑 A pluggable pipeline (receivers → processors → exporters) that decouples your app from any specific backend.

Source: https://opentelemetry.io/docs/collector/

## Pipeline Anatomy
| Stage | Purpose | Examples |
|---|---|---|
| **Receivers** | Ingest signals | `otlp`, `prometheus`, `filelog`, `jaeger`, `zipkin` |
| **Processors** | Transform/buffer/sample | `batch`, `memory_limiter`, `tail_sampling`, `attributes`, `resource` |
| **Exporters** | Ship to backends | `otlphttp`, `prometheusremotewrite`, `loki`, `sentry`, vendor-specific |
| **Connectors** | Pipe one pipeline into another | `spanmetrics` (traces → metrics) |
| **Extensions** | Side features | `health_check`, `pprof`, `zpages` |

## Two Distros
- **Core** — minimal, mainline components only.
- **Contrib** — kitchen sink (Loki, Tempo, vendor exporters, fluent receiver, etc.). Use this in practice.

## Minimum-Viable Config
```yaml
receivers:
  otlp:
    protocols: { grpc: {}, http: {} }

processors:
  memory_limiter: { check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 25 }
  batch: { timeout: 5s, send_batch_size: 8192 }

exporters:
  otlphttp/tempo: { endpoint: http://tempo:4318 }
  prometheusremotewrite: { endpoint: http://prom:9090/api/v1/write }

service:
  pipelines:
    traces:  { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlphttp/tempo] }
    metrics: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [prometheusremotewrite] }
```

## Critical Processors
- `memory_limiter` **first** — drops data before OOM.
- `batch` **last** — coalesces network calls.
- `tail_sampling` — full-trace decisions; see [[Sampling]].

## Deployment Modes
- **Agent** (per-host/sidecar) — collects local, ships to gateway.
- **Gateway** (cluster) — central fan-in, tail sampling, auth, multi-tenant.

## Tags
[[OpenTelemetry]] [[Collector]] [[OTLP]] [[Sampling]]
