# 02 — Use Cases

🔑 Kafka's partitioned, replicated, durable log is general-purpose enough to back messaging, telemetry, log aggregation, stream processing, event sourcing, and DB-style commit logs.

Source: https://kafka.apache.org/42/getting-started/uses/

## Use Case Map
| Use Case | What It Solves | Why Kafka Fits |
|---|---|---|
| **Messaging** | Decouple producers from processors; buffer unprocessed work | Better throughput, built-in partitioning, replication, fault-tolerance vs ActiveMQ/RabbitMQ |
| **Website Activity Tracking** | Real-time pub/sub feed for page views, searches, clicks | Topic-per-activity-type; subscribers do real-time monitoring or batch warehousing |
| **Metrics** | Aggregate operational stats from distributed apps | Centralized operational data feed |
| **Log Aggregation** | Replace file-based aggregators (Scribe, Flume) | Equal perf, stronger durability via replication, lower end-to-end latency |
| **Stream Processing** | Multi-stage pipelines: raw → enriched → aggregated | Kafka Streams (≥0.10.0.0) is lightweight; alternatives: Storm, Samza |
| **Event Sourcing** | Persist state changes as time-ordered log | Supports very large stored log data |
| **Commit Log** | External commit log for distributed systems; node re-sync | Log compaction enables data sync and failed-node recovery (cf. Apache BookKeeper) |

## Notes Per Use Case

### Messaging
- ⚠️ Volume per use case is typically lower than other workloads, but latency + durability requirements are strict.
- Replaces traditional message brokers when you need horizontal scale.

### Website Activity Tracking
- 💡 Original Kafka use case at LinkedIn.
- High volume — many events per page view.

### Log Aggregation
- Abstracts files into a clean event-stream consumer model.
- Multiple sources, distributed consumption.

### Stream Processing
- Build DAGs of topics where each stage transforms upstream data.
- Kafka Streams is in-process; no separate cluster.

### Event Sourcing
- Application design pattern — state = fold over event log.
- Kafka's long retention + compaction makes this practical.

### Commit Log
- Use Kafka as the replication backbone for another distributed system.
- Compacted topics = key-keyed snapshot, suitable for re-syncing nodes.

## Tags
[[Kafka]] [[Event Streaming]] [[Topics]] [[Replication]] [[Streams]] [[Connect]]
