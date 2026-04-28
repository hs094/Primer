# 03 — Rebalancing

🔑 A rebalance is the protocol where the group coordinator reassigns partitions across group members. Modern Kafka uses the **cooperative sticky** assignor (incremental, no stop-the-world) and **static membership** (`group.instance.id`) to skip rebalances during rolling restarts.

Source: https://kafka.apache.org/42/configuration/consumer-configs/

## Triggers
- Member joins (new consumer subscribes).
- Member leaves cleanly (`consumer.close()`) or via failure detection.
- Subscription changes (e.g. regex matches a new topic).
- Partition count changes on a subscribed topic.

## Eager vs Cooperative
| | Eager (`RangeAssignor`, legacy `StickyAssignor`) | Cooperative (`CooperativeStickyAssignor`, default) |
|---|---|---|
| Protocol | Stop-the-world: every member revokes **all** partitions, then rejoins | Only reassigned partitions are revoked; the rest keep flowing |
| Pause | Whole group halts during rebalance | Per-partition, brief |
| Migration | One step | Two steps (revoke, then assign) |

⚠️ Modern default is `RangeAssignor, CooperativeStickyAssignor`. For production set explicitly to `[CooperativeStickyAssignor]` and migrate carefully — eager and cooperative cannot coexist in one group.

## Static membership
- Set `group.instance.id="worker-3"` (must be unique per pod/replica) — typically the StatefulSet ordinal or hostname.
- Coordinator now identifies the consumer by `group.instance.id` instead of the ephemeral session.
- A clean restart within `session.timeout.ms` rejoins **without rebalancing** — partitions stay put.
- Use for stateful consumers, large in-memory caches, or anywhere rebalance is expensive.

## Rebalance listener
Hook into revoke/assign to flush state and seek:
```python
from confluent_kafka import Consumer

def on_revoke(consumer, partitions) -> None:
    flush_local_state()
    consumer.commit(asynchronous=False)

def on_assign(consumer, partitions) -> None:
    restore_local_state(partitions)
    # confluent-kafka: call assign() here if you want manual control
    consumer.assign(partitions)

consumer.subscribe(["orders"], on_revoke=on_revoke, on_assign=on_assign)
```
- `aiokafka`: subclass `ConsumerRebalanceListener` with `on_partitions_revoked` / `on_partitions_assigned`.

## Heartbeats and liveness
| Config | Default | Meaning |
|---|---|---|
| `heartbeat.interval.ms` | `3000` | Background heartbeat cadence — keeps session alive |
| `session.timeout.ms` | `45000` | No heartbeat for this long ⇒ coordinator considers member dead |
| `max.poll.interval.ms` | `300000` (5 min) | Time between `poll()` calls — exceeding this ejects you (slow processing detection) |

Rule: `heartbeat.interval.ms` ≤ `session.timeout.ms / 3`.

⚠️ `max.poll.interval.ms` and `session.timeout.ms` detect different failures: heartbeat thread runs in background even if your processing thread stalls — `max.poll.interval.ms` catches the latter.

💡 Lower `max.poll.records` if processing each batch takes long; otherwise raise `max.poll.interval.ms`.

🧪 Try-it: start 3 consumers with `group.instance.id` set, kill one, restart it within 30s — observe no partition reassignment in `kafka-consumer-groups --describe`.

## Tags
[[Kafka]] [[Consumers]] [[Consumer Groups]] [[Rebalance]]
