# 01 — Overview

🔑 Kafka is a distributed event streaming platform: publish/subscribe + durable log + stream processing, all on a partitioned, replicated commit log.

Source: https://kafka.apache.org/42/getting-started/introduction/

## Event Streaming
- Capture real-time event streams from sources (DBs, sensors, apps).
- Store durably for replay; process now or later; route to sinks.
- "Central nervous system" pattern for data-driven systems.

## What Kafka Is
| Capability | Meaning |
|---|---|
| **Publish/subscribe** | Read and write event streams; continuous import/export to/from external systems |
| **Durable storage** | Retain events for as long as configured retention demands |
| **Stream processing** | Process events as they arrive or retrospectively |

Runs distributed, fault-tolerant, elastic, secure — bare metal, VMs, containers, on-prem, cloud.

## How It Works
- **Servers**: brokers form the storage cluster; Kafka Connect runs alongside for I/O integration. Cluster is fault-tolerant — failed brokers fail over automatically.
- **Clients**: read/write/process at scale. Available in Java, Scala, Go, Python, C/C++, plus REST.
- TCP wire protocol; clients and servers communicate via that protocol.

## Core Concepts
| Concept | Definition |
|---|---|
| **Event / record** | Key, value, timestamp, optional headers. The unit of data. |
| **Producer** | Client that publishes events. |
| **Consumer** | Client that subscribes to events. Decoupled from producers. |
| **Topic** | Named, ordered, durable log of events. Like a folder. Multi-producer, multi-subscriber. |
| **Partition** | Topic shard living on a broker. Same key → same partition. |
| **Offset** | Per-partition position. Consumers track progress via offsets. |
| **Replication** | Partitions copied across brokers. Production: factor of 3. |
| **Retention** | Events persist by config — they don't vanish on consume. |

⚠️ Producers and consumers are fully decoupled — never call each other.

💡 Per-partition order is the hard guarantee. Cross-partition order is not.

## The 5 APIs
| API | Use |
|---|---|
| **Admin** | Manage/inspect topics, brokers, ACLs, configs |
| **Producer** | Publish event streams to topics |
| **Consumer** | Subscribe to topics and process events |
| **Streams** | Transform/aggregate/window/join streams (stateful) |
| **Connect** | Reusable source/sink connectors to external systems (hundreds exist) |

## Tags
[[Kafka]] [[Event Streaming]] [[Topics]] [[Partitions]] [[Brokers]] [[Producers]] [[Consumers]] [[Replication]]
