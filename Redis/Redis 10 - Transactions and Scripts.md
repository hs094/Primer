# 10 — Transactions and Scripts

🔑 `MULTI`/`EXEC` queues commands and runs them as one atomic unit; `WATCH` adds optimistic concurrency; Lua `EVAL` is the same atomicity with branching logic.

Source: https://redis.io/docs/latest/develop/interact/transactions/

## MULTI / EXEC / DISCARD
```
MULTI
SET balance:a 90
SET balance:b 110
EXEC
```
- Commands between `MULTI` and `EXEC` are **queued**, not run.
- `EXEC` runs the whole batch atomically — no other client sees a partial state.
- `DISCARD` throws the queue away.
- ⚠️ Redis txns do **not** roll back on a runtime error — already-executed commands stand. Only syntax errors before `EXEC` abort.

## WATCH — Optimistic Locking
```python
async with r.pipeline(transaction=True) as pipe:
    while True:
        try:
            await pipe.watch("balance:a")
            cur = int(await r.get("balance:a"))
            if cur < 100:
                await pipe.unwatch()
                break
            pipe.multi()
            pipe.decrby("balance:a", 100)
            pipe.incrby("balance:b", 100)
            await pipe.execute()      # None if WATCHed key changed → retry
            break
        except redis.WatchError:
            continue
```
- `WATCH k…` → if any watched key is modified by anyone before `EXEC`, the transaction aborts (returns nil).
- 💡 Pattern for read-modify-write without a server-side lock.

## Lua via EVAL
```python
LUA = """
local v = redis.call('GET', KEYS[1])
if v == ARGV[1] then return redis.call('DEL', KEYS[1]) end
return 0
"""
await r.eval(LUA, 1, "lock:job", token)
```
- Whole script runs atomically, single-threaded — no other command interleaves.
- Use `EVALSHA` after `SCRIPT LOAD` to skip re-sending the body.
- ⚠️ Long scripts block the server. Keep them short; `lua-time-limit` defaults to 5 s before SLOWLOG warnings.
- Functions (`FUNCTION LOAD`, ≥7.0) are the modern, persisted, named alternative.

## Pipeline vs Transaction
| | Pipeline | MULTI/EXEC |
|---|---|---|
| Bytes batched in one TCP round-trip | ✅ | ✅ (built on pipelining) |
| Atomic / no interleaving | ❌ | ✅ |
| Conditional logic | ❌ | Only via WATCH |
| Branching / loops | ❌ | Use Lua |

- 💡 Pipeline = throughput. Transaction = correctness. Lua = both, when you need branching.

## Tags
[[Redis]] [[Transactions]] [[Lua]] [[Pipelines]] [[Locks]]
