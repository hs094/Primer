# 10 — Delivery Semantics and Transactions

🔑 Kafka offers at-most-once / at-least-once / exactly-once delivery — exactly-once is built from the **idempotent producer** plus **transactions**, both available since 0.11.0.0. 4.x adds **share groups** for queue-style consumption.

Source: https://kafka.apache.org/42/design/design/

## Three Delivery Modes
| Mode | Producer behavior | Consumer behavior |
|---|---|---|
| At-most-once | Don't retry on ambiguous ack | Commit offset before processing |
| At-least-once | Retry until acked | Commit offset after processing |
| Exactly-once | Idempotent + transactional producer | `read_committed`, commit offsets in same txn |

## Definition of "Committed"
- A message is committed once **all replicas in the current ISR** have applied it.
- Only committed messages are exposed to consumers.
- Producers using `acks=all` block until commit; weaker `acks` trade durability for latency.

## Idempotent Producer
- Enabled with `enable.idempotence=true` (default in modern clients).
- Each producer gets a **PID** (producer ID) and per-partition **sequence number**.
- Broker tracks `(PID, partition) → last seq`; rejects duplicates and out-of-order writes.
- Eliminates duplicates from network retries — without transactions, but only within one producer session.
- Cost: tiny — sequence number per record, per-batch state on broker.

## Transactions
- Atomic writes across multiple partitions / topics (and offset commits).
- Required producer config:
  - `transactional.id` — stable across restarts; lets new instance fence the old one (zombie producer protection).
- Required consumer config:
  - `isolation.level=read_committed` — skips records inside aborted transactions.
  - `enable.auto.commit=false` — offsets committed via `producer.sendOffsetsToTransaction()`.
- Control batches (commit/abort markers) flag txn outcome; consumers use them to filter aborted records.
- Use case: read-process-write loops where input offset commit and output writes must be atomic.

## Transaction Exception Hierarchy
| Class | Meaning |
|---|---|
| `RetriableException` | Transient; retry the operation |
| `RefreshRetriableException` | Retry after refreshing metadata |
| `AbortableException` | Abort the transaction, can start a new one |
| `ApplicationRecoverableException` | App must reinitialize the producer |

## Share Groups (NEW in 4.x)
A queue-style consumption model that complements consumer groups.

| Aspect | Consumer Group | Share Group |
|---|---|---|
| Partition → consumer | 1:1 within group | Many consumers can share one partition |
| Consumer count vs partitions | Capped at # partitions | Can exceed partitions |
| Ack granularity | Per offset (commit) | Per record |
| Delivery model | Pull, offset-based | Pull, lock-based |

### Lock-Based Delivery
- Default lock duration: `share.record.lock.duration.ms = 30s`.
- A delivered record is locked to that consumer until it acks or the lock expires.
- `group.share.partition.max.record.locks` caps in-flight locked records per partition.

### Acks
| Ack | Effect |
|---|---|
| `accept` | Mark record done; not redelivered |
| `release` | Unlock immediately for redelivery to anyone |
| `reject` | Permanently skip (poison message) |
| `renew` | Extend the lock |

- 💡 Share groups make Kafka usable as a work queue without sacrificing the shared log — same topic can feed both classic consumer groups and share groups.

## Static Membership
- `GROUP_INSTANCE_ID_CONFIG` (brokers 2.3+) gives a consumer a stable identity.
- Rolling restarts no longer trigger full rebalance — broker recognizes the returning member by its instance id and preserves its assignment.
- ⚠️ If a static member crashes hard, partitions stay assigned to the dead consumer until session timeout — pick the timeout to match expected restart window.

## Tags
[[Kafka]] [[Delivery Semantics]] [[Idempotent Producer]] [[Transactions]] [[Share Groups]] [[Consumers]] [[Producers]] [[Brokers]]
