# 01 — Overview

🔑 Redis is an in-memory data structure store — single-threaded command loop, RESP wire protocol, rich types beyond plain k/v.

Source: https://redis.io/docs/latest/get-started/

## What Redis Is
| Aspect | Detail |
|---|---|
| **Storage** | In-memory; optional disk persistence (RDB/AOF) |
| **Model** | Single-threaded event loop for command execution |
| **Protocol** | RESP (REdis Serialization Protocol) over TCP :6379 |
| **Types** | Strings, lists, hashes, sets, sorted sets, streams, JSON, bitmaps, HLL, geo, vectors |

## Single-Threaded Model
- One core executes commands sequentially → every op is atomic by construction.
- I/O threads (≥6.0) parallelize socket read/write only; command logic stays serial.
- ⚠️ A slow command (`KEYS *`, big `LRANGE`) blocks the whole server. Use `SCAN`, paginate ranges.

## RESP Protocol
- Line-based, binary-safe. Easy to implement → tons of clients.
- `SET foo bar` on the wire: `*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`.
- RESP3 (≥6.0) adds richer types (maps, sets, doubles, push messages for client-side caching).

## Why People Reach For It
- **Cache** in front of slow DBs.
- **Counters / rate limits** via `INCR` + TTL.
- **Queues / streams** via lists or [[Streams]].
- **Pub/sub & realtime** fanout.
- **Vector search & semantic cache** with Redis Stack / [[RedisVL]].

## 💡 Hello, redis-py async
```python
import redis.asyncio as redis

r = redis.from_url("redis://localhost:6379", decode_responses=True)
await r.set("hello", "world", ex=60)
print(await r.get("hello"))
await r.aclose()
```

## Tags
[[Redis]] [[RESP]] [[Streams]] [[RedisVL]]
