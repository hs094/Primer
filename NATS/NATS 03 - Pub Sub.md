# 03 — Pub/Sub

🔑 Core NATS [[Pub Sub]] is fire-and-forget, at-most-once — no broker-side persistence, no offsets, no replay.

Source: https://docs.nats.io/nats-concepts/core-nats/pubsub

## Semantics
- Publisher sends to a subject; server fans out to **all currently-subscribed** clients.
- ⚠️ If no subscriber is connected when you publish, the message is **dropped silently**.
- No acks at the Core layer — for durability use [[JetStream]].

## Python (`nats-py`, asyncio)
```python
import asyncio
import nats

async def main() -> None:
    nc = await nats.connect("nats://localhost:4222")

    async def on_msg(msg) -> None:
        print(f"{msg.subject}: {msg.data.decode()}")

    sub = await nc.subscribe("orders.created.>", cb=on_msg)
    await nc.publish("orders.created.us", b'{"id": 123}')

    await asyncio.sleep(0.1)
    await sub.unsubscribe()
    await nc.drain()

asyncio.run(main())
```

## vs [[Kafka]]
| | NATS Core | Kafka |
|---|---|---|
| Persistence | ❌ ephemeral | ✅ durable log |
| Offsets / replay | ❌ | ✅ |
| Subscribe latency | µs | ms (poll loop) |
| Fan-out cost | cheap (broadcast) | per-consumer-group disk reads |
| Use case | live signals, RPC | event sourcing, ETL |

💡 Reach for NATS Core when you want a *bus*, Kafka when you want a *log*. Use [[JetStream]] when you want both.

## Tags
[[NATS]] [[Pub Sub]] [[Messaging]] [[Kafka]]
