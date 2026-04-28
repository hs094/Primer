# 04 — Log Compaction

🔑 Compaction keeps the *latest* value per key forever, instead of expiring by time or size — turning a topic into a queryable state journal.

Source: https://kafka.apache.org/42/design/design/

## Use Cases
- **Change Data Capture (CDC)** — mirror a table where the latest row per PK is what matters.
- **Event sourcing** — replay full state by reading from offset 0.
- **Journal-as-state** — system-of-record stored as an append-only log with the log itself doubling as the materialized view.
- **Internal**: `__consumer_offsets` and many KRaft metadata topics use compaction.

## Mechanics
- Log split into **head** (recent, may have duplicates) and **tail** (already cleaned).
- Cleaner thread pool runs in the background per partition.
- Cleaning a partition:
  1. Build an offset map of `key → latest offset` for the head — 24 bytes per entry hash table.
  2. Recopy the log front-to-back, skipping any record whose offset is older than the latest for its key.
- Compaction does not block reads or writes — uses copy-on-write segment list.

## Buffer Sizing
- 💡 An 8 GB cleaner buffer can clean ~366 GB of log head when records are 1 KB.
- Tune `log.cleaner.dedupe.buffer.size` if you have lots of large partitions or small records.

## Tombstones
- A record with `value = null` is a **tombstone** — signals deletion of that key.
- Compaction keeps the tombstone, then drops it after `delete.retention.ms` (default 24h).
- Consumers must see the tombstone within that window to learn the key is gone.
- ⚠️ Set `delete.retention.ms` ≥ your max consumer downtime.

## Key Configs
| Config | Purpose |
|---|---|
| `log.cleanup.policy=compact` | Enable compaction for the topic |
| `log.cleanup.policy=compact,delete` | Compact + still apply time/size retention |
| `log.cleaner.min.compaction.lag.ms` | Don't compact records younger than this |
| `log.cleaner.max.compaction.lag.ms` | Force compaction after this even if buffer isn't full |
| `min.cleanable.dirty.ratio` | Trigger threshold (dirty / total bytes) |
| `delete.retention.ms` | Tombstone visibility window |

## Guarantees
- A consumer reading from offset 0 sees every key's latest value at least once.
- Ordering within a key is preserved (older versions removed, newer kept in original order).
- Offset positions of surviving records are **unchanged**.

## Tags
[[Kafka]] [[Log Compaction]] [[Tombstone]] [[Event Sourcing]] [[CDC]] [[Brokers]]
