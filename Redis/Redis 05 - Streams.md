# 05 — Streams

🔑 Append-only log baked into Redis: monotonically-increasing IDs, consumer groups with per-message ack, and pending-entry recovery.

Source: https://redis.io/docs/latest/develop/data-types/streams/

## Core Commands
| Command | Effect |
|---|---|
| `XADD stream * field value …` | Append entry; `*` = server-assigned `ms-seq` ID |
| `XLEN stream` | Length |
| `XRANGE stream - +` / `XREVRANGE` | Iterate by ID range |
| `XREAD COUNT n BLOCK ms STREAMS s id` | Tail one or more streams (no group) |
| `XGROUP CREATE s grp $ MKSTREAM` | Create consumer group from "now" |
| `XREADGROUP GROUP grp consumer COUNT n BLOCK ms STREAMS s >` | Group read — `>` = new entries only |
| `XACK s grp id` | Mark delivered |
| `XPENDING s grp` / `XCLAIM` / `XAUTOCLAIM` | Inspect / steal stuck PEL entries |
| `XTRIM s MAXLEN ~ N` / `MINID ms-seq` | Cap stream size |

## Consumer Group Semantics
- Each entry is delivered to exactly one consumer in the group.
- Until `XACK`, the entry sits in the consumer's **PEL** (pending entries list).
- Crashed consumer? Another can `XAUTOCLAIM` and resume — at-least-once delivery.
- ⚠️ Idempotency is your job — duplicates are possible across claim/redeliver.

## Producer / Consumer (redis-py async)
```python
# producer
await r.xadd("orders", {"user": uid, "total": str(total)}, maxlen=100_000, approximate=True)

# consumer
await r.xgroup_create("orders", "billing", id="$", mkstream=True)  # once
while True:
    resp = await r.xreadgroup("billing", "worker-1", {"orders": ">"}, count=10, block=5000)
    for stream, msgs in resp or []:
        for msg_id, fields in msgs:
            await handle(fields)
            await r.xack("orders", "billing", msg_id)
```

## Streams vs Kafka
| Aspect | Redis Streams | Kafka |
|---|---|---|
| Ordering | Per stream (single shard) | Per partition |
| Throughput ceiling | Single-node bound | Horizontal via partitions |
| Retention | `MAXLEN` / `MINID` cap | Time/size, log-segment based |
| Consumer model | Group + PEL + XACK | Group + offsets + commit |
| Ops cost | One Redis | Kafka + ZK/KRaft + schema reg |
| Sweet spot | < ~1M msg/s, low-latency, already running Redis | Multi-MB/s, multi-DC, long retention |

## Tags
[[Redis]] [[Streams]] [[Consumer Group]] [[Kafka]]
