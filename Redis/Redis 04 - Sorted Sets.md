# 04 — Sorted Sets

🔑 Sets where every member carries a float score — O(log N) insert, range-by-score in O(log N + M). The leaderboard / time-window primitive.

Source: https://redis.io/docs/latest/develop/data-types/sorted-sets/

## Core Commands
| Command | Effect |
|---|---|
| `ZADD k score member [score member …]` | Add / update score |
| `ZADD k GT score m` / `LT` | Only update if greater / lesser |
| `ZINCRBY k delta member` | Bump score atomically |
| `ZSCORE k m` / `ZRANK k m` | Score / rank lookup |
| `ZRANGE k 0 9 REV WITHSCORES` | Top 10 by score desc |
| `ZRANGEBYSCORE k min max LIMIT off cnt` | Score window (deprecated alias for `ZRANGE … BYSCORE`) |
| `ZREMRANGEBYSCORE k -inf t` | Trim old entries |
| `ZPOPMIN k [n]` / `ZPOPMAX k [n]` | Pop lowest / highest |

## Leaderboard
```python
await r.zadd("leaderboard:season1", {user_id: score})
top10 = await r.zrange("leaderboard:season1", 0, 9, desc=True, withscores=True)
my_rank = await r.zrevrank("leaderboard:season1", user_id)  # 0-indexed
```

## Time-Series via score=timestamp
```python
ts = time.time()
await r.zadd(f"events:{user_id}", {event_json: ts})
# last 24h:
since = ts - 86400
await r.zrangebyscore(f"events:{user_id}", since, "+inf")
# evict older than 7d:
await r.zremrangebyscore(f"events:{user_id}", "-inf", ts - 7*86400)
```
- 💡 Cheap rolling window without Streams when you only need range queries.
- ⚠️ Member must be unique. If the same payload arrives twice, you only get one entry — append a uniquifier (`f"{ts}:{uuid4()}"`) when needed.

## Priority Queue with ZPOPMIN
```python
await r.zadd("jobs", {payload: due_ts})
job = await r.bzpopmin("jobs", timeout=5)  # blocks for next-due job
```

## Rate Limiting (sliding log)
- Add request timestamps to a ZSET, trim entries older than `window`, count remaining → reject if over limit.

## Tags
[[Redis]] [[Sorted Sets]] [[Leaderboard]] [[Rate Limit]]
