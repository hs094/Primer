# 03 — Group and Streams Configs

🔑 Server-side per-group-type configs (consumer/share/streams) plus client-side Kafka Streams configs. Group configs are dynamic and applied to specific group ids.

Sources:
- https://kafka.apache.org/42/configuration/group-configs/
- https://kafka.apache.org/42/configuration/kafka-streams-configs/

## Group Configs (server-side, per group)

These override broker defaults like `group.consumer.session.timeout.ms`, scoped to a single `group.id`. Set with `kafka-configs.sh --entity-type groups`.

### Consumer groups (KIP-848 protocol)

| key | type | default | meaning |
| --- | --- | --- | --- |
| `consumer.session.timeout.ms` | int | 45000 (45s) | Failure-detection timeout. Member evicted if no heartbeat in this window. |
| `consumer.heartbeat.interval.ms` | int | 5000 (5s) | How often members heartbeat. Should be ≪ session timeout (≈ 1/3). |

### Share groups (KIP-932, queue-like semantics)

| key | type | default | meaning |
| --- | --- | --- | --- |
| `share.session.timeout.ms` | int | 45000 (45s) | Member-failure detection for share groups. |
| `share.heartbeat.interval.ms` | int | 5000 (5s) | Share-group member heartbeat cadence. |
| `share.record.lock.duration.ms` | int | 30000 (30s) | How long a fetched record stays locked to a member before re-delivery. |
| `share.isolation.level` | string | `read_uncommitted` | `read_uncommitted` or `read_committed` for transactional records. |
| `share.auto.offset.reset` | string | `latest` | `earliest` or `latest` — where new share-partitions start. |

### Streams groups (KIP-1071)

| key | type | default | meaning |
| --- | --- | --- | --- |
| `streams.session.timeout.ms` | int | 45000 (45s) | Streams member failure detection. |
| `streams.heartbeat.interval.ms` | int | 5000 (5s) | Streams group heartbeat cadence. |
| `streams.num.standby.replicas` | int | 0 | Server-side standby count override for the group. |
| `streams.initial.rebalance.delay.ms` | int | 3000 (3s) | Wait for more members to join on first startup before rebalancing. |

⚠️ `consumer.*`, `share.*`, `streams.*` here are **group-level overrides**. The broker-level versions are `group.consumer.session.timeout.ms`, etc. — same semantics, broader scope. Min/max bounds (`group.consumer.min.session.timeout.ms`, `group.consumer.max.session.timeout.ms`) are broker-only.

## Kafka Streams Configs (client-side)

| key | type | default | meaning |
| --- | --- | --- | --- |
| `application.id` | string | (required) | Identity of the Streams app. Drives consumer group id, internal topic prefix, state-dir layout. |
| `bootstrap.servers` | list | (required) | Initial broker connection list. |
| `processing.guarantee` | string | `at_least_once` | `at_least_once` or `exactly_once_v2` (transactional, RF≥3). |
| `num.stream.threads` | int | 1 | StreamThread count per instance. Bump for higher parallelism (capped by total partition count). |
| `replication.factor` | int | -1 | RF for changelog/repartition topics. `-1` = use broker default. Production: 3. |
| `state.dir` | string | `${java.io.tmpdir}` | Local state-store directory. ⚠️ Tmp is wiped on reboot — set to durable mount in prod. |
| `default.key.serde` | class | null | Default key Serde. |
| `default.value.serde` | class | null | Default value Serde. |
| `cache.max.bytes.buffering` | long | 10485760 (10 MiB) | Total record-cache memory across threads. ⚠️ Deprecated in favour of `statestore.cache.max.bytes`. |
| `statestore.cache.max.bytes` | long | 10485760 (10 MiB) | Statestore cache memory. Larger = more dedup before downstream emit. |
| `commit.interval.ms` | long | 30000 (30s) | Offset/changelog commit cadence. With `exactly_once_v2`, default drops to 100ms. |
| `num.standby.replicas` | int | 0 | Hot standby count per task — speeds failover at the cost of disk + network. |
| `group.protocol` | string | `classic` | `classic` or `streams` (KIP-1071 broker-driven). |
| `max.task.idle.ms` | long | 0 | Wait time for a task with stale partitions before joining results out-of-order. |
| `task.timeout.ms` | long | 300000 (5m) | Task-stall budget before throwing `TaskCorruptedException`. |
| `topology.optimization` | string | `none` | `none`, `all`, or specific opts (e.g. `reuse.ktable.source.topics`). |

💡 EOS recipe: `processing.guarantee=exactly_once_v2`, RF≥3 brokers, `min.insync.replicas=2`, idempotent producers (default in 3.x+), and `replication.factor=3` for Streams internal topics.

🧪 Try it: list a streams app's internal topics with `kafka-topics.sh --list | grep <application.id>`.

## Tags
[[Kafka]] [[Streams]] [[Brokers]]
