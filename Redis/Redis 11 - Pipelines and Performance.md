# 11 — Pipelines and Performance

🔑 Latency is RTT-bound on small commands — pipelining batches N requests into one round-trip and 10x's throughput on remote networks.

Source: https://redis.io/docs/latest/develop/use/pipelining/

## The Math
- Each command without pipelining = 1 RTT.
- 1 ms RTT × 10 000 commands = 10 s wall clock, even on a beefy server.
- Pipeline depth = N → wall ≈ `commands / N × RTT + server_time`.

## redis-py Async Pipeline
```python
async with r.pipeline(transaction=False) as pipe:
    for i in range(10_000):
        pipe.set(f"k:{i}", i)
    await pipe.execute()  # one network turn
```
- `transaction=False` → plain pipeline (just batching).
- `transaction=True` → wraps in `MULTI/EXEC` (adds atomicity, costs a tiny bit).

## Pipeline vs Transaction (revisited)
| | Pipeline | Transaction |
|---|---|---|
| Reduces RTT | ✅ | ✅ (uses pipelining) |
| Atomic | ❌ | ✅ |
| Other clients can interleave | ✅ | ❌ |
| Use when | Bulk ops, throughput | Correctness across keys |

## When Pipelining Wins (and Doesn't)
- ✅ Remote / cross-AZ Redis where RTT ≥ 0.5 ms.
- ✅ Bulk loads, fan-out writes, scan-and-process.
- ❌ Loopback / Unix socket — RTT is already negligible; CPU/memory dominate.
- ⚠️ Keep pipeline batches bounded (e.g. 1k-10k commands). Multi-MB pipelines blow client + server buffers.
- ⚠️ Pipelined writes may all be in flight when a connection drops — partial application possible. Use Lua or `MULTI/EXEC` if atomicity matters.

## Benchmarking with redis-benchmark
```bash
# 1M GETs, 50 concurrent clients, no pipeline
redis-benchmark -h host -p 6379 -n 1000000 -c 50 -t get

# Same workload with pipeline depth 16
redis-benchmark -h host -p 6379 -n 1000000 -c 50 -P 16 -t get

# Realistic key sizes
redis-benchmark -t set,get -d 1024 -n 500000
```
- `-P` is the pipeline lever — sweep 1, 4, 16, 64 to find the knee.
- `-r N` randomizes keyspace (avoids hot-cache distortion).
- ⚠️ `redis-benchmark` is a synthetic loop — your app's bottleneck is usually the client lib + serialization, not Redis.

## 💡 Other Wins
- Use `MGET` / `HMGET` / `SMISMEMBER` — server-side batching beats N round-trips.
- Connection pool size ≈ concurrent in-flight ops; over-pooling burns memory, under-pooling queues requests in-process.
- `CLIENT NO-EVICT on`, `latency doctor`, `SLOWLOG GET 10` for prod triage.

## Tags
[[Redis]] [[Pipelines]] [[Performance]] [[Benchmarking]]
