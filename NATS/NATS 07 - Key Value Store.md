# 07 — Key/Value Store

🔑 KV is a thin map API on top of a [[JetStream]] stream — every put is a message, the latest value per key is the "current" state.

Source: https://docs.nats.io/nats-concepts/jetstream/key-value-store

## Model
- A **bucket** is a stream named `KV_<bucket>` with subject pattern `$KV.<bucket>.<key>`.
- `put(k, v)` publishes; `get(k)` reads the latest message for that key.
- **History per key** (default 1, max 64) keeps last N revisions for audit/rewind.
- **TTL** on bucket auto-expires entries — useful for sessions, leases.

## Atomic Ops
- `create(k, v)` — fails if key exists (compare-to-null).
- `update(k, v, last_revision)` — CAS; fails if revision moved.
- `purge(k)` — tombstone, drops history.

## Python
```python
kv = await js.create_key_value(bucket="config", history=5, ttl=3600)

await kv.put("feature.dark_mode", b"on")
entry = await kv.get("feature.dark_mode")
print(entry.value, entry.revision)

# Watch for live changes
async for update in await kv.watchall():
    print(update.key, update.value, update.operation)
```

## vs [[Redis]]
| | NATS KV | Redis |
|---|---|---|
| Persistence | JetStream replicated log | RDB/AOF snapshot |
| Watch / pub-sub | built-in, per-key | Keyspace notifications (lossy) |
| History / revisions | ✅ N versions | ❌ |
| Data structures | bytes only | rich (lists, sets, streams, …) |
| Latency | sub-ms | sub-ms |

💡 Reach for NATS KV when you already run NATS and need durable config / coordination. Reach for Redis for rich types and high QPS lookups.

## Tags
[[NATS]] [[JetStream]] [[Redis]]
