# 07 — Messages and Record Batch

🔑 Records are always written as part of a **record batch** — the batch is the unit of compression, transactionality, and producer idempotency.

Source: https://kafka.apache.org/42/implementation/messages/, https://kafka.apache.org/42/implementation/message-format/

## Message Anatomy
- Variable-length header + opaque key bytes + opaque value bytes.
- Key and value are deliberately byte-arrays — Kafka takes no opinion on serialization (Avro, Protobuf, JSON, etc. live above).
- `RecordBatch` is an iterator over messages with bulk read/write on an NIO channel.

## RecordBatch On-Disk Format (magic = 2)
```
baseOffset:           int64
batchLength:          int32
partitionLeaderEpoch: int32
magic:                int8   // current = 2
crc:                  uint32 // CRC-32C (Castagnoli)
attributes:           int16
lastOffsetDelta:      int32
baseTimestamp:        int64
maxTimestamp:         int64
producerId:           int64
producerEpoch:        int16
baseSequence:         int32
recordsCount:         int32
records:              [Record]
```
- Total size on disk = `batchLength + 12` (the 8-byte `baseOffset` + 4-byte `batchLength` itself).
- CRC covers attributes → end of batch. Located **after** the magic byte so clients parse magic first to know how to interpret bytes between length and magic.
- `partitionLeaderEpoch` is excluded from CRC — broker sets it on receive without recomputing.

## Attributes Bits
| Bit | Meaning |
|---|---|
| 0–2 | Compression: 0 none, 1 gzip, 2 snappy, 3 lz4, 4 zstd |
| 3 | Timestamp type |
| 4 | `isTransactional` |
| 5 | `isControlBatch` |
| 6 | `hasDeleteHorizonMs` (compaction tombstone marker) |
| 7–15 | Unused |

## Record Format (inside batch)
```
length:        varint
attributes:    int8        // currently unused
timestampDelta: varlong    // delta from baseTimestamp
offsetDelta:   varint      // delta from baseOffset
keyLength:     varint
key:           byte[]
valueLength:   varint
value:         byte[]
headersCount:  varint
headers:       [Header]
```
- Varint encoding (Protobuf-style) saves bytes on common small values.
- Timestamps and offsets stored as deltas → tighter packing for batches.

## Record Headers
- `headerKey: String` (non-null), `headerValue: byte[]` (nullable).
- Order preserved through produce/consume.
- Use cases: trace IDs, schema versions, content type, routing hints.

## Compaction Quirks
- Compaction preserves **first and last** offset/sequence numbers from each batch — needed to restore producer state and prevent `OutOfSequence` after leader failover.
- 💡 An "empty" batch can survive compaction if all its records were cleaned but its sequence range is still load-bearing.
- `baseTimestamp` is **not** preserved through compaction; can shift when the first record is cleaned away.
- For null-payload records or aborted-txn markers, `baseTimestamp` is reset to the delete-horizon time and bit 6 flips.

## Control Batches
- Single record per batch, not delivered to applications.
- Control record key:
  - `version: int16` (currently 0)
  - `type: int16` — `0 = abort marker`, `1 = commit`
- Used by consumers (`isolation.level=read_committed`) to filter aborted transactional records.
- Value schema is type-dependent and opaque to clients.

## Old Message Format
- ⚠️ Pre-0.11, messages were transferred and stored as "message sets" without batch headers, idempotency state, or transaction markers. Anything modern (idempotent producer, transactions, compaction-safe sequencing) requires magic 2.

## Tags
[[Kafka]] [[Record Batch]] [[Message Format]] [[Idempotent Producer]] [[Transactions]] [[Brokers]]
