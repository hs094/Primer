# 02 — Acks and Idempotence

🔑 `acks` controls durability; `enable.idempotence=true` (now the default) gives in-order, dedup-on-retry, exactly-once-per-partition writes by stamping each record with a Producer ID and per-partition sequence number.

Source: https://kafka.apache.org/42/configuration/producer-configs/

## Acks semantics
| acks | Wait for | Durability | Latency | Notes |
|---|---|---|---|---|
| `0` | nothing | weakest — fire-and-forget | lowest | record may be lost on broker crash; no retry on transport error |
| `1` | leader only | medium | low | lost if leader crashes before replicating |
| `all` (`-1`) | leader + all in-sync replicas | strongest | highest | required for idempotence; honors `min.insync.replicas` |

⚠️ `acks=all` alone does **not** prevent duplicates — only guarantees the write is replicated. Idempotence handles the dedup half.

## Idempotent producer
- `enable.idempotence=true` (default since Kafka 3.0).
- Broker assigns a **Producer ID (PID)** on first produce; producer keeps a **per-partition sequence number** that increments per message.
- Broker dedupes by `(PID, partition, seq)` — a retry of a previously-written record returns success without writing twice.
- Detects out-of-order writes — broker rejects gaps, producer recovers.

### Required config
- `acks=all` (forced)
- `retries > 0` (default `Integer.MAX_VALUE`)
- `max.in.flight.requests.per.connection <= 5` (otherwise ordering not guaranteed)

If you set conflicting values, the client throws `ConfigException` at startup.

## Exactly-once scope
- Idempotence = exactly once **per partition, per producer session**.
- A new `Producer` instance gets a new PID — duplicates across restarts are still possible.
- For cross-partition, cross-session atomicity, use [[Transactions]].

## Comparison
| Mode | Duplicates? | Order? | Across restart? |
|---|---|---|---|
| `acks=0/1`, no idempotence | yes (on retry) | maybe | no |
| `acks=all`, no idempotence | yes (on retry) | yes | no |
| `acks=all` + idempotence | **no** (within session) | yes | no |
| Transactions | no | yes | **yes** (via `transactional.id`) |

💡 Idempotence is free safety in modern Kafka — leave it on unless you have hard latency reasons.

## Tags
[[Kafka]] [[Producers]] [[Idempotent Producer]] [[Transactions]]
