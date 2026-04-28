# 01 — Kafka Streams Overview

🔑 A client library that turns ordinary Java/Scala apps into stream processors — no separate cluster to operate.

Source: https://kafka.apache.org/42/streams/

## What it is
- **Library, not a framework.** You add `kafka-streams` to your JAR and run it like any other Kafka client.
- **No separate processing cluster.** Parallelism comes from input topic partitions; scale by launching more app instances in the same `application.id` group.
- **Just Kafka.** Brokers handle coordination, fault-tolerance, and storage of state changelogs — there is no JobManager, ResourceManager, or Driver.

## Pipeline shape
```
input topics  →  topology (source → processors → sink)  →  output topics
```
- Source nodes consume from one or more input topics.
- Processor nodes do the work (filter, map, aggregate, join, window, …).
- Sink nodes produce to output topics.
- State stores (RocksDB or in-memory) sit beside processors; their changelog is itself a Kafka topic.

## Streams vs Spark / Flink

| Aspect | [[Kafka Streams]] | Spark Structured Streaming / Flink |
| --- | --- | --- |
| Deployment | Library inside your app | Dedicated cluster (driver/executors, JobManager/TaskManager) |
| Resource manager | None — JVMs + Kafka | YARN, Kubernetes, standalone |
| Source/sink | Kafka only | Many (Kafka, files, JDBC, sockets, …) |
| State | Local RocksDB + Kafka changelog | Distributed state backend (RocksDB + checkpoints to DFS) |
| Latency | Per-record, ms | Micro-batch (Spark) or per-record (Flink) |
| Scale unit | App instance × input partitions | Cluster slots / executors |

💡 Pick Streams when your data already lives in Kafka and you want the simplest possible operational story.
⚠️ Streams is JVM only — no native Python. For Python, use Connect + a downstream consumer or `confluent-kafka` + Faust-style processing.

## Tags
[[Kafka]] [[Kafka Streams]] [[KStream]] [[KTable]]
