# 02 — OpenTelemetry Overview

🔑 OTel is a vendor-neutral standard: one API to instrument, one wire protocol (OTLP) to ship, any backend to receive.

Source: https://opentelemetry.io/docs/

## The Stack
| Layer | Role |
|---|---|
| **API** | What app code calls (`tracer.start_span`, `meter.create_counter`). Stable, no I/O. |
| **SDK** | Reference impl: providers, processors, exporters, samplers. |
| **Instrumentation libraries** | Auto-spans for FastAPI, httpx, SQLAlchemy, Redis, etc. |
| **OTLP** | Wire protocol (gRPC or HTTP/protobuf). One format for all signals. |
| **Collector** | Receive → process → export pipeline (separate process). |

## Three Signals
- **Traces** — stable.
- **Metrics** — stable.
- **Logs** — stable spec, SDK maturity varies by language.

## Why Vendor-Neutral Matters
- Instrument once; swap backend (Jaeger → Tempo → Datadog) by changing exporter config only.
- 90+ vendors support OTLP natively — no rip-and-replace risk.

## Data Flow (typical)
```
app (API+SDK) ──OTLP──> Collector ──> Tempo / Prometheus / Loki / Sentry / SaaS
```

## Resource Attributes
- Every signal carries a `Resource` (service.name, service.version, deployment.environment, k8s.pod.name…).
- 💡 Set `OTEL_RESOURCE_ATTRIBUTES=service.name=api,service.version=1.4.2` once per process.

## Tags
[[OpenTelemetry]] [[OTLP]] [[Collector]] [[Traces]] [[Metrics]] [[Logs]]
