# 01 — Overview

🔑 [[NATS]] is a connective layer for distributed systems: subject-based pub/sub with sub-millisecond latency over a tiny text protocol.

Source: https://docs.nats.io/nats-concepts/overview

## What It Is
- Single binary (`nats-server`), written in Go, no external deps.
- **Subject-based addressing** replaces hostnames/ports — services discover each other by name, not location.
- M:N connectivity by default; no load balancers, proxies, or sidecars needed for fan-out.

## Performance Profile
| Trait | Detail |
|---|---|
| Latency | Sub-ms in-cluster; tens of ms cross-region |
| Throughput | Millions of msgs/sec per node |
| Payload | 1 MB default, up to 64 MB configurable |
| Protocol | Plain-text, line-oriented (`PUB`, `SUB`, `MSG`, `PING`) — easy to debug with `telnet` |

## Use Cases
- 💡 **Microservice [[Messaging]]** — request/reply RPC, event fan-out, queue-group workers.
- 💡 **IoT / [[Edge]]** — leaf nodes bridge devices to a central cluster over flaky links.
- 💡 **Observability + telemetry** — high-cardinality event streams to analytics sinks.
- 💡 **Command & control** — fleet management, OTA pushes, real-time dashboards.

## Why Not Just [[Kafka]]?
- NATS Core is **fire-and-forget** (at-most-once); Kafka is durable log by design.
- For durability, NATS adds [[JetStream]] — but the Core layer stays cheap.
- NATS topology heals dynamically (gossip); Kafka cluster membership is static-ish (ZooKeeper/KRaft).

## Tags
[[NATS]] [[Messaging]] [[Edge]] [[Pub Sub]]
