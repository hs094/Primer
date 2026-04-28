# 01 — Broker Configs

🔑 The knobs that define a broker's identity, networking, durability, and KRaft role. Most are static (require restart); some are `cluster-wide` and dynamic via `kafka-configs.sh`.

Source: https://kafka.apache.org/42/configuration/broker-configs/

## Identity & KRaft Roles

| key | type | default | meaning |
| --- | --- | --- | --- |
| `broker.id` | int | -1 | Broker id. `-1` = auto-generate. Prefer `node.id` in KRaft mode. |
| `node.id` | int | (required) | KRaft node id; required when `process.roles` is set. |
| `process.roles` | list | (empty) | `broker`, `controller`, or `broker,controller` (combined mode). Empty = legacy ZK mode (removed in 4.x → must set). |
| `controller.listener.names` | list | (required) | Listener name(s) the controller quorum talks on. Broker uses the first. |
| `controller.quorum.bootstrap.servers` | list | "" | `host:port,...` endpoints for bootstrapping cluster metadata (preferred over `controller.quorum.voters` since 3.7). |
| `controller.quorum.voters` | list | "" | Static voter map: `{id}@{host}:{port},...`. |

⚠️ KRaft is mandatory in 4.x — no ZooKeeper. You must define `process.roles`, `node.id`, `controller.listener.names`, and a quorum.

## Networking

| key | type | default | meaning |
| --- | --- | --- | --- |
| `listeners` | list | `PLAINTEXT://:9092` | URIs the broker binds to: `LISTENER_NAME://host:port,...`. |
| `advertised.listeners` | list | null | What clients/peers connect to. Set this when behind NAT/k8s. Falls back to `listeners`. |
| `num.network.threads` | int | 3 | Acceptor/processor threads handling socket I/O. Bump for high client counts. |
| `num.io.threads` | int | 8 | Request-handler threads doing disk + network work. Should be ≥ disk count. |
| `socket.send.buffer.bytes` | int | 102400 | SO_SNDBUF. `-1` uses OS default. |
| `socket.receive.buffer.bytes` | int | 102400 | SO_RCVBUF. `-1` uses OS default. |

## Storage & Logs

| key | type | default | meaning |
| --- | --- | --- | --- |
| `log.dirs` | list | null | Comma-separated log directories. Use multiple paths for JBOD spread across disks. Falls back to `log.dir` (`/tmp/kafka-logs`). |
| `log.segment.bytes` | int | 1 GiB | Max size per segment file before roll. |
| `log.retention.ms` | long | null | Retention window. If null, fall back to `log.retention.minutes` → `log.retention.hours` (default 168 = 7 days). |
| `log.retention.bytes` | long | -1 | Per-partition size cap. `-1` = unlimited. |
| `log.cleanup.policy` | list | `delete` | `delete`, `compact`, or both. Cluster default; topics override. |

## Replication & Durability

| key | type | default | meaning |
| --- | --- | --- | --- |
| `default.replication.factor` | int | 1 | RF for auto-created topics. ⚠️ Bump to 3 in production. |
| `num.partitions` | int | 1 | Default partition count for auto-created topics. |
| `min.insync.replicas` | int | 1 | Min ISR required for `acks=all` writes to succeed. Production: set to 2 with RF=3. |
| `unclean.leader.election.enable` | boolean | false | Allow out-of-ISR replicas to be elected leader. ⚠️ True trades durability for availability — keep false. |
| `replica.lag.time.max.ms` | long | 30000 | Follower kicked from ISR if behind for this long. |
| `auto.create.topics.enable` | boolean | true | Cluster default lets producers/consumers auto-create topics. ⚠️ Most production deployments set this `false` — explicit topic creation only. |
| `delete.topic.enable` | boolean | true | Allow `kafka-topics --delete`. |

## Transactions & Group Coordination

| key | type | default | meaning |
| --- | --- | --- | --- |
| `transaction.state.log.replication.factor` | short | 3 | RF of `__transaction_state`. ⚠️ Cluster needs ≥3 brokers before EOS works. |
| `transaction.state.log.min.isr` | int | 2 | Min ISR for `__transaction_state` writes. |
| `group.coordinator.rebalance.protocols` | list | `classic,consumer,streams` | Enabled rebalance protocols. KIP-848 `consumer` (broker-driven) and `streams` are now defaults. |

💡 Production baseline: `RF=3`, `min.insync.replicas=2`, `acks=all` from producers, `unclean.leader.election.enable=false`. This is the durable-by-default trio.

🧪 Try it: dump effective broker configs with `kafka-configs.sh --bootstrap-server :9092 --entity-type brokers --entity-name 1 --all --describe`.

## Tags
[[Kafka]] [[Brokers]] [[KRaft]]
