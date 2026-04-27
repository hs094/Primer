# SH 08 — Monitoring

🔑 **Server and SDKs emit Prometheus-compatible metrics by default. Watch sync match rate, schedule-to-start latency, and persistence latency — they reveal saturation before workflows back up.**

## Server Metrics (Prometheus)

Temporal Service exposes `/metrics` on a configured port. Sample scrape config:

```yaml
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'temporalmetrics'
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets:
          - 'host.docker.internal:8000'
        labels:
          group: 'server-metrics'
      - targets:
          - 'host.docker.internal:8077'
          - 'host.docker.internal:8078'
        labels:
          group: 'sdk-metrics'
```

Boot the dev server with metrics:

```bash
temporal server start-dev --metrics-port 8000
```

Verify scraping at `http://localhost:9090/targets`.

## SDK Metrics

SDK metrics are configured **in worker / client code** — Go, Java, Python, TypeScript, .NET, PHP, Ruby all expose a metrics handler that you wire to Prometheus on a port of your choosing. The worker itself opens that port; Prometheus scrapes it.

Ports must match `prometheus.yml` targets. Each worker process needs its own scrape target (or use service discovery).

## Grafana

```bash
docker run -d -p 3000:3000 grafana/grafana-enterprise
```

Add Prometheus data source: `http://host.docker.internal:9090`. Default login `admin` / `admin`.

Import community dashboards from `github.com/temporalio/dashboards` — pre-built panels for server and SDK.

## Key Metrics To Watch

### Service Health

| Metric | What it means |
|---|---|
| `temporal_request_latency` (per service) | Frontend / history / matching gRPC latency p50/p99/p999. |
| `temporal_request_failure` | Per-service error count by gRPC code. |
| `temporal_persistence_latency` | Direct DB latency view. Spikes mean DB saturation. |
| `temporal_persistence_errors` | DB connection / write failures. |

### Workflow Execution Health

| Metric | Target |
|---|---|
| `temporal_sync_match_rate` | **> 99%** — fraction of tasks dispatched without polling. Lower = workers starved. |
| `temporal_schedule_to_start_latency` | Time tasks wait for a worker. High = scale workers. |
| `temporal_workflow_completed` / `_failed` / `_canceled` / `_timeout` | Outcome distribution. |
| `temporal_activity_*` | Activity outcomes & latencies. |

### Persistence

| Metric | Watch for |
|---|---|
| Read/write latency p99/p999 | DB saturation. |
| Connection pool saturation | Pool exhaustion → throttled writes. |
| Shard movement | Frequent reassignment = instability. |

⚠️ **Sync match rate is your single best lagging indicator of worker starvation.** If it drops, schedule-to-start latency rises, workflow throughput collapses, and tasks queue in the matching service.

## Health Checks

Frontend supports TCP and gRPC health checks on `:7233`. Wire to k8s liveness / readiness probes, Nomad checks, or your LB.

```yaml
livenessProbe:
  tcpSocket:
    port: 7233
  initialDelaySeconds: 30
  periodSeconds: 10
```

## Alternative Stacks

- **Datadog** — direct integration for [[Temporal TC 01 - Cloud Overview|Temporal Cloud]]; for self-hosted run Datadog Agent with the Prometheus check pointed at the metrics endpoint.
- **OpenTelemetry Collector** — scrape the `/metrics` endpoint and forward to any OTLP backend.
- **Helm charts** — pre-configured monitoring sidecars and ServiceMonitor CRDs.

## Alerting Starter Set

- Frontend gRPC error rate > 1% over 5m.
- `temporal_persistence_latency` p99 > SLA × 2 over 5m.
- `temporal_sync_match_rate` < 95% over 5m.
- `temporal_schedule_to_start_latency` p99 above your activity SLO.
- Worker pollers = 0 on a critical task queue.
- Workflow `_failed` rate spike (vs baseline).

⚠️ Without metrics in place pre-launch, Temporal failures look like "the system is slow" — workflows back up silently in matching while persistence ticks along fine.

💡 **Takeaway:** Scrape `/metrics` from server and every worker, dashboard the community Grafana JSON, and alert on sync match rate / schedule-to-start / persistence latency. If those three are green, workflows are flowing.
