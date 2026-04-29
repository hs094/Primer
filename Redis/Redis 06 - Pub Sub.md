# 06 — Pub/Sub

🔑 Fire-and-forget broadcast: publishers shout into a channel, only currently-connected subscribers hear it. Zero persistence.

Source: https://redis.io/docs/latest/develop/interact/pubsub/

## Core Commands
| Command | Effect |
|---|---|
| `PUBLISH ch msg` | Send; returns # subscribers reached |
| `SUBSCRIBE ch1 ch2 …` | Listen on exact channels |
| `PSUBSCRIBE news.*` | Glob-pattern subscribe |
| `UNSUBSCRIBE` / `PUNSUBSCRIBE` | Stop |
| `PUBSUB CHANNELS [pattern]` / `PUBSUB NUMSUB` | Introspect |
| `SPUBLISH` / `SSUBSCRIBE` | Sharded pub/sub for [[Redis Cluster]] (≥7.0) |

## redis-py Async
```python
pubsub = r.pubsub()
await pubsub.subscribe("orders.created")
async for msg in pubsub.listen():
    if msg["type"] == "message":
        handle(msg["data"])
```

## Semantics — Read This Twice
- **No persistence.** A message published with zero subscribers is dropped — forever.
- **No replay.** A subscriber that joins late never sees prior messages.
- **No ack.** Slow consumers are killed when their output buffer overflows (`client-output-buffer-limit pubsub`).
- ⚠️ Need durability / replay / consumer groups? Use [[Streams]], not pub/sub.
- 💡 Pub/sub is great for cache invalidation fanout, dashboards, transient signaling.

## Cluster Caveat
- Plain `PUBLISH` fans out across all nodes (cluster bus) — fine for low rates.
- `SPUBLISH` is per-shard, scales with cluster size, but subscribers must hit the right shard.

## Keyspace Notifications
Redis can publish to internal channels when keys change.
```bash
CONFIG SET notify-keyspace-events KEA
SUBSCRIBE __keyevent@0__:expired
```
- `K` = keyspace events, `E` = keyevent, `A` = all event classes.
- 💡 Use to react to TTL expiry (e.g. session timeout) without polling.
- ⚠️ Notifications use pub/sub — same fire-and-forget rules. If your subscriber is down, you miss events.

## Tags
[[Redis]] [[Pub Sub]] [[Streams]] [[Keyspace Notifications]]
