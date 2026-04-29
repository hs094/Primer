# 02 — Strings and Counters

🔑 Strings are byte blobs up to 512 MB — but the real superpower is atomic numeric ops + TTL for caches, counters, and locks.

Source: https://redis.io/docs/latest/develop/data-types/strings/

## Core Commands
| Command | Effect |
|---|---|
| `SET k v [EX s] [PX ms] [NX] [XX]` | Set value, optional TTL, only-if-(not-)exists |
| `GET k` | Read |
| `INCR k` / `DECR k` | Atomic +1 / -1 (creates as 0) |
| `INCRBY k n` / `INCRBYFLOAT k f` | Atomic +n (int / float) |
| `APPEND k s` | Append, returns new length |
| `MSET k1 v1 k2 v2` / `MGET k1 k2` | Multi-key batch |
| `GETDEL`, `GETEX` | Read + delete / read + reset TTL |

## TTL Patterns
- `SET session:abc <jwt> EX 3600` — 1-hour cache.
- `EXPIRE k 60` / `PEXPIRE k 60000` after the fact.
- `TTL k` → seconds left, `-1` no expiry, `-2` missing.
- ⚠️ TTL drift on overwrite: `SET k v` (no `KEEPTTL`) wipes expiry. Use `SET k v KEEPTTL` to preserve.

## Atomic Counters
```python
hits = await r.incr(f"hits:{path}")
await r.expire(f"hits:{path}", 86400, nx=True)  # set TTL only first time
```
- `INCR` is the canonical hot-counter primitive — single-threaded server = no race.
- Sliding window via `INCR` + bucketed key (`hits:2026-04-28:13`).

## SETNX as a Lock (with caveats)
```python
ok = await r.set("lock:job", token, nx=True, ex=30)
if ok:
    try: ...
    finally:
        # Lua compare-and-del to avoid releasing someone else's lock
        await r.eval("if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) end",
                     1, "lock:job", token)
```
- ⚠️ Single-node SETNX is **not** safe under failover — for that, see RedLock or a real coordinator.
- 💡 Always `EX` a lock so a crashed holder doesn't deadlock.

## Tags
[[Redis]] [[RESP]] [[Strings]] [[TTL]] [[Locks]]
