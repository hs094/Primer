# 01 — Kafka Connect Overview

🔑 A separate JVM cluster that moves data **into** and **out of** Kafka via pluggable connectors, managed over REST.

Source: https://kafka.apache.org/42/kafka-connect/overview/

## What it is
- A framework for **scalable, reliable** data integration with Kafka.
- You don't write producers/consumers — you configure a connector class and Connect handles partitioning, parallelism, offsets, retries, and rebalancing.
- Two flavors of connector:
  - **Source connector** — pulls from external system → Kafka topic. (e.g. Debezium MySQL, JDBC, S3 source)
  - **Sink connector** — pushes from Kafka topic → external system. (e.g. Elasticsearch, JDBC, S3 sink, BigQuery)

## Runtime
- Runs in its own **JVM cluster of "workers"**, separate from Kafka brokers.
- Connect itself is a Kafka client — it does *not* run on the brokers.
- Workers form a group via Kafka's group-coordination protocol; tasks are balanced across workers.

## Modes

| Mode | When to use | Coordination |
| --- | --- | --- |
| **Standalone** | Dev / single-machine / log shipper at the edge | All state in local files |
| **Distributed** | Production / multi-worker / fault-tolerant | State in three Kafka topics (`config.storage.topic`, `offset.storage.topic`, `status.storage.topic`) |

⚠️ Don't mix modes in the same `group.id`. Standalone configs use file-backed state; distributed needs the storage topics.

## REST API
- Workers expose a REST port (default `8083`) — same API on every worker, requests are forwarded to the leader as needed.
- Submit / inspect / pause / restart / delete connectors here. (See [[Kafka Connect 02 - User Guide]].)

## Pluggable converters
- Converter = serde for Kafka record key/value at the Connect boundary (Connect's internal model is `Schema` + `Struct`).

| Converter class | Format |
| --- | --- |
| `JsonConverter` | JSON, optional embedded schema |
| `StringConverter` | Plain UTF-8 |
| `ByteArrayConverter` | Pass-through bytes |
| `AvroConverter` (Confluent) | Avro + [[Schema Registry]] |
| `ProtobufConverter` (Confluent) | Protobuf + [[Schema Registry]] |
| `JsonSchemaConverter` (Confluent) | JSON Schema + [[Schema Registry]] |

💡 Pair Avro/Protobuf converters with a [[Schema Registry]] for schema evolution; otherwise inline JSON schemas balloon payloads.

## Tags
[[Kafka]] [[Kafka Connect]] [[Connector]] [[Schema Registry]]
