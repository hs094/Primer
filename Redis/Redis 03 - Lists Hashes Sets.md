# 03 — Lists, Hashes, Sets

🔑 Three workhorse collections: lists for queues/timelines, hashes for record-shaped values, sets for membership and set algebra.

Source: https://redis.io/docs/latest/develop/data-types/

## Lists (doubly-linked, O(1) ends)
| Command | Use |
|---|---|
| `LPUSH/RPUSH k v…` | Push left/right |
| `LPOP/RPOP k [n]` | Pop |
| `BLPOP/BRPOP k timeout` | Blocking pop — simple work queue |
| `LRANGE k 0 -1` | Read range |
| `LLEN k` / `LTRIM k 0 999` | Size / cap timeline |

```python
await r.lpush("queue:emails", payload)
job = await r.brpop("queue:emails", timeout=5)  # returns (key, value) or None
```

## Hashes (field → value within one key)
| Command | Use |
|---|---|
| `HSET k f v [f v …]` | Set fields |
| `HGET k f` / `HMGET k f1 f2` | Read |
| `HGETALL k` | All fields (small hashes only) |
| `HINCRBY k f n` | Atomic per-field counter |
| `HEXPIRE k seconds FIELDS n f1…` (≥7.4) | Per-field TTL |

- 💡 Use a hash instead of N separate string keys for "object-ish" data — less memory, one round-trip.
- ⚠️ `HGETALL` on a 100k-field hash is a blocking command. Use `HSCAN`.

## Sets (unique, unordered)
| Command | Use |
|---|---|
| `SADD k m…` / `SREM k m` | Add / remove |
| `SISMEMBER k m` / `SMISMEMBER k m1 m2` | Membership |
| `SCARD k` | Count |
| `SINTER`, `SUNION`, `SDIFF` | Set algebra across keys |
| `SRANDMEMBER`, `SPOP` | Random sampling |

```python
await r.sadd(f"user:{uid}:tags", "python", "redis")
common = await r.sinter(f"user:{a}:tags", f"user:{b}:tags")
```

## Picking The Right Type
| Need | Type |
|---|---|
| FIFO queue, recent-N timeline | List |
| Object with many fields | Hash |
| Tags / unique ids / dedup | Set |
| Ranked / scored | [[Sorted Sets]] (next note) |

## Tags
[[Redis]] [[Lists]] [[Hashes]] [[Sets]]
