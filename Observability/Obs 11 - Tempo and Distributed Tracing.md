# 11 — Tempo & Distributed Tracing

🔑 Cheap trace store: object storage + trace-ID lookup, no full-text index. Pair with metrics (exemplars) and logs (trace_id) for the three-way pivot.

Source: https://grafana.com/docs/tempo/latest/

## What Tempo Is
- Distributed tracing backend; receives OTLP / Jaeger / Zipkin.
- Storage: **object storage only** (S3, GCS, Azure Blob). No expensive Cassandra/ES.
- Optimized for **trace ID lookup** + TraceQL search.

## Distributed Tracing Refresher
- **Span** = one unit of work (one DB call, one HTTP handler).
- **Trace** = DAG of spans sharing `trace_id`.
- Context propagated in headers: `traceparent`, `tracestate` (W3C Trace Context).

## The Three-Way Pivot
```
metric panel ── exemplar ──> trace ── trace_id ──> logs
        ▲                                            │
        └──────────── service.name label ────────────┘
```
- **Exemplars**: Prometheus stores a sample `trace_id` alongside histogram buckets. Click point → jump to slow trace.
- **Service graph**: Tempo's metrics-generator reads spans, emits `traces_service_graph_request_total` etc. → live topology in Grafana.

## TraceQL (essentials)
```
{ resource.service.name = "api" && status = error }
{ duration > 1s && span.http.route = "/checkout" }
{ name = "POST /pay" } | avg(duration) > 500ms
```

## Operational Notes
- ⚠️ Tempo can't grep span content; pre-filter via [[Sampling]] (tail-sample slow + errors).
- 💡 Enable metrics-generator → free RED dashboards (rate, errors, duration) from your traces.
- ⚠️ Retention is set on the storage backend (S3 lifecycle), not Tempo. 7–30 days typical.

## Tags
[[Tempo]] [[Traces]] [[OpenTelemetry]] [[Grafana]] [[Exemplars]]
