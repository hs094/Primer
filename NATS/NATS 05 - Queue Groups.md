# 05 — Queue Groups

🔑 Subscribers sharing a queue group on the same subject get **one-of-N** delivery — instant, server-side load balancing.

Source: https://docs.nats.io/nats-concepts/core-nats/queue

## Mechanics
- Subscriber registers with `(subject, queue_group)` instead of just `subject`.
- For each message on `subject`, the server picks **one** member of the queue group at random.
- Plain (non-queue) subscribers on the same subject still get every message in parallel.

```python
# Three workers, same queue group → each msg goes to exactly one
await nc.subscribe("jobs.image.resize", queue="resizers", cb=worker)
```

## Composition
- Subject + queue is the unit. `jobs.> / "workers-A"` and `jobs.> / "workers-B"` are independent groups.
- Scale by **starting more processes** with the same queue name; shrink by stopping them. No config change.
- 💡 Cluster routing is **geo-affine** — local queue members are preferred, cutting cross-region hops.

## vs [[Kafka]] Consumer Groups
| | NATS queue group | Kafka consumer group |
|---|---|---|
| Offsets | ❌ ephemeral | ✅ committed per partition |
| Rebalance | implicit, instant | explicit, stop-the-world |
| Replay | ❌ (use [[JetStream]]) | ✅ |
| Partition count | irrelevant | hard cap on parallelism |
| Member churn cost | zero | seconds (rebalance storm) |

⚠️ Without [[JetStream]], a crashed worker mid-message **drops** that message. For at-least-once, use a JetStream pull consumer instead.

## Tags
[[NATS]] [[Pub Sub]] [[Messaging]] [[Kafka]]
