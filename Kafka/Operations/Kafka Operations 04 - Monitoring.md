# 04 — Monitoring

🔑 Kafka exposes everything via **JMX**. Scrape with JMX Exporter → Prometheus → Grafana.

Source: https://kafka.apache.org/42/operations/monitoring/

## Critical Broker Metrics

| Metric | Healthy | Meaning |
|---|---|---|
| `UnderReplicatedPartitions` | **0** | Partitions where ISR < replicas. Non-zero → degraded. |
| `OfflinePartitionsCount` | **0** | Unavailable partitions. Page-worthy. |
| `ActiveControllerCount` | **1 cluster-wide** | Exactly one broker should report 1. |
| `RequestHandlerAvgIdlePercent` | > **0.3** | I/O thread headroom. Low → request thread starvation. |
| `NetworkProcessorAvgIdlePercent` | > **0.3** | Network thread headroom. |

⚠️ Sum `ActiveControllerCount` across brokers — alert on `!= 1`.

## Producer Metrics

| Metric | Watch |
|---|---|
| `record-error-rate` | Sustained > 0 → broker/auth/quota issue. |
| `request-latency-avg` | Spikes → broker pressure or network. |
| `record-send-rate` | Throughput sanity. |
| `compression-rate-avg` | Verify compression actually working. |

## Consumer Metrics

| Metric | Watch |
|---|---|
| `records-lag-max` | Primary SLO signal — max lag across assigned partitions. |
| `fetch-rate` | Fetches/sec. |
| `fetch-latency-avg` | Broker fetch latency. |
| `commit-rate` | Offset-commit cadence. |

💡 `records-lag-max` is per-consumer; aggregate group-wide lag externally (Burrow).

## Stacks

| Stack | Use |
|---|---|
| **JMX Exporter + Prometheus + Grafana** | OSS default. Scrape JMX MBeans into Prom. |
| **Burrow** | Dedicated consumer-lag monitor with sliding-window evaluation; no static thresholds. |
| **Confluent Control Center** | Commercial UI, end-to-end. |
| **Cruise Control** | Workload balance + anomaly detection. |

## Access

```bash
# point at running broker
jconsole localhost:9999
```

⚠️ Remote JMX in production needs auth + TLS, otherwise it's a free RCE vector.

## Tags
[[Kafka]] [[Operations]]
