# 06 — JetStream

🔑 [[JetStream]] is the persistence layer bolted onto Core NATS — streams store messages, consumers read them with acks.

Source: https://docs.nats.io/nats-concepts/jetstream

## Streams + Consumers
- **Stream** = durable, ordered log of messages matching a set of subjects (`orders.>`).
- **Consumer** = cursor over the stream, with delivery + ack policy.
- Decoupled: many consumers per stream, each with their own position.

## Retention Policies
| Policy | Removes message when | Use case |
|---|---|---|
| **Limits** (default) | Age / size / count cap hit | Event log, replayable history |
| **Interest** | No active consumer cares | Pub/sub with safety net |
| **WorkQueue** | Any consumer acks it | Job queue, exactly-one worker |

⚠️ WorkQueue allows **only one** consumer subscription per subject filter — pick it deliberately.

## Delivery Guarantees
- **At-least-once** by default (publisher gets PubAck; consumer must `ack()` or msg redelivers).
- **Exactly-once** via per-message dedup IDs (`Nats-Msg-Id` header, dedup window on stream) + double-ack on consume.

## Python
```python
js = nc.jetstream()

# Producer side: server replies with ack
ack = await js.publish("orders.created.us", b"...", headers={"Nats-Msg-Id": "ord-123"})
print(ack.stream, ack.seq)

# Pull consumer (durable, scalable)
sub = await js.pull_subscribe("orders.created.>", durable="billing")
msgs = await sub.fetch(10, timeout=2.0)
for m in msgs:
    process(m)
    await m.ack()
```

💡 Use **pull consumers** for workers (back-pressure, horizontal scale); **push** for live UI feeds.

## Tags
[[NATS]] [[JetStream]] [[Messaging]] [[Kafka]]
