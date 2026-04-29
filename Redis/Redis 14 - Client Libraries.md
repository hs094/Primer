# 14 — Client Libraries

🔑 Stick to the language's officially-supported client; reuse a single connection pool per process; pick async if your app is async.

Source: https://redis.io/docs/latest/develop/clients/

## The Short List
| Language | Client | Notes |
|---|---|---|
| Python | `redis-py` | Sync + async (`redis.asyncio`) in one package |
| Python (AI) | `RedisVL` | Built on redis-py; vectors / semantic cache |
| Node.js | `node-redis` | Official, modern; `ioredis` is the popular legacy alt |
| Java | `Lettuce` (async/reactive) or `Jedis` (sync) | Lettuce thread-safe, Netty-based |
| Go | `go-redis` | Idiomatic, supports cluster/sentinel |
| .NET | `StackExchange.Redis` | Multiplexed connection model |
| Rust | `redis-rs` (`fred` for async) | |
| C | `hiredis` | Thin RESP wrapper, foundation for many bindings |

## redis-py Async (canonical)
```python
import redis.asyncio as redis

# from URL — most ergonomic
r = redis.from_url(
    "redis://:password@localhost:6379/0",
    decode_responses=True,
    max_connections=50,
    socket_timeout=2.0,
    socket_keepalive=True,
    health_check_interval=30,
)

await r.set("k", "v", ex=60)
val = await r.get("k")
await r.aclose()
```

## ioredis (Node)
```js
import Redis from "ioredis";
const r = new Redis({ host: "localhost", port: 6379, maxRetriesPerRequest: 3 });
await r.set("k", "v", "EX", 60);
```

## go-redis
```go
rdb := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
    PoolSize: 50,
    MinIdleConns: 5,
    DialTimeout: 2 * time.Second,
})
defer rdb.Close()
ctx := context.Background()
_ = rdb.Set(ctx, "k", "v", time.Minute).Err()
```

## Connection Pool — Defaults That Bite
| Knob | Default-ish | Pick |
|---|---|---|
| `max_connections` / `PoolSize` | 10–50 | ≈ concurrent in-flight ops; not request count |
| `socket_timeout` | unlimited (!) | 1–5 s — avoid hangs on a hung Redis |
| `socket_keepalive` | off | on, especially behind NAT/LB |
| `health_check_interval` | off | 30–60 s for long-lived idle conns |
| `retry_on_timeout` / retry policy | off | Enable with backoff for transient cluster blips |

- ⚠️ One pool per process, not per request. Creating a client per call destroys throughput.
- 💡 For [[Redis Cluster]], use the cluster-aware client (`RedisCluster`, `redis.Cluster`, `ClusterClient`) — it caches the slot map and follows `MOVED`/`ASK` automatically.
- ⚠️ `decode_responses=True` returns `str`; for binary (vectors, protobuf), keep it `False` and decode selectively.

## FastAPI Pattern
```python
# lifespan
app.state.redis = redis.from_url(settings.redis_url, max_connections=50)

# dependency
async def get_redis() -> redis.Redis:
    return app.state.redis  # one shared pool
```

## Tags
[[Redis]] [[redis-py]] [[ioredis]] [[go-redis]] [[Connection Pool]]
