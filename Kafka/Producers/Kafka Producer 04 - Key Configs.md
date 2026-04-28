# 04 — Key Producer Configs

🔑 The producer config knobs you actually tune. Defaults are sensible in 4.x — touch these only when you know why.

Source: https://kafka.apache.org/42/configuration/producer-configs/

## Reference table
| Config | Default | What it does |
|---|---|---|
| `bootstrap.servers` | (required) | Comma-separated `host:port` list for initial cluster discovery. |
| `acks` | `all` | Replicas that must ack: `0` (none), `1` (leader), `all` (leader + ISR). |
| `retries` | `2147483647` | Retry count for transient send failures; bounded by `delivery.timeout.ms`. |
| `batch.size` | `16384` (16 KiB) | Max bytes per partition batch before flush. Bigger = more compression, more latency. |
| `linger.ms` | `5` | Wait this long for more records before sending a partial batch. Tune up for throughput. |
| `compression.type` | `none` | `none`/`gzip`/`snappy`/`lz4`/`zstd`. `lz4`/`zstd` good defaults. Applied per-batch. |
| `max.in.flight.requests.per.connection` | `5` | Unacked requests per broker connection. Must be ≤5 with idempotence to keep order. |
| `enable.idempotence` | `true` | Dedup retries via PID + seq; forces `acks=all`. Leave on. |
| `transactional.id` | `null` | Stable id for cross-batch atomic commits + zombie fencing. See [[Transactions]]. |
| `transaction.timeout.ms` | `60000` | Coordinator aborts a stuck txn after this. |
| `delivery.timeout.ms` | `120000` | Hard upper bound from `produce()` to ack/error. ≥ `linger.ms + request.timeout.ms`. |
| `request.timeout.ms` | `30000` | Per-request broker response timeout; triggers retry. |
| `buffer.memory` | `33554432` (32 MiB) | Total client-side buffer. `produce()` blocks (or fails after `max.block.ms`) when full. |
| `max.block.ms` | `60000` | How long `produce()` blocks on full buffer or unknown metadata before throwing. |
| `key.serializer` / `value.serializer` | (required, Java) | Class turning objects → `bytes`. In `confluent-kafka` Python, pass bytes directly. |
| `partitioner.class` | `null` (built-in sticky/Murmur2) | Custom `Partitioner` impl. Almost always leave default. |
| `client.id` | `""` | Free-form id surfaced in broker logs/metrics. Set it — eases debugging. |

## Tuning cheatsheet
- **High throughput, tolerable latency**: `linger.ms=20-50`, `batch.size=131072`, `compression.type=lz4`.
- **Low latency**: `linger.ms=0`, smaller `batch.size`, but you lose batching wins.
- **Strong durability**: leave defaults (`acks=all`, idempotence on) and ensure broker `min.insync.replicas>=2`.

⚠️ `delivery.timeout.ms` must be ≥ `linger.ms + request.timeout.ms`, else producer rejects config.

💡 Don't disable idempotence for "perf" — its cost is negligible on modern brokers and you lose dedup safety.

## Tags
[[Kafka]] [[Producers]] [[Idempotent Producer]] [[Transactions]]
