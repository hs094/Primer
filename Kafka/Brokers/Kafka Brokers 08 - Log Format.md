# 08 — Log Format

🔑 A partition is a directory of segment files named by base offset; reads do a binary search over an in-memory segment range, then scan within the segment.

Source: https://kafka.apache.org/42/implementation/log/

## Directory Layout
- Topic `my-topic` with 2 partitions → directories `my-topic-0/` and `my-topic-1/`.
- Each partition directory contains a sequence of segment files.
- Each log file is named by the **offset of its first message**, zero-padded:
  - `00000000000000000000.log` (first segment)
  - `00000000000010485760.log` (next, etc.)

## Segment Files
- `.log` — the actual record batches.
- `.index` — sparse offset → file-position index (binary search target).
- `.timeindex` — sparse timestamp → offset index (used for offset-by-time lookups).
- 💡 Segments are also tracked with leader-epoch and producer snapshot files for recovery.

## On-Disk Entry
- Sequence of "log entries": `4-byte length N` + `N message bytes`.
- Each message is uniquely identified by its **64-bit offset** (byte position within the partition's full message stream).
- Format is versioned via the magic byte so producers, brokers, and clients can transfer batches **without recopy or conversion** when versions match.

## Why Offsets, Not GUIDs
- Original idea: producer-generated GUID + global mapping. Rejected because:
  - Each broker would need a heavyweight persistent random-access index.
  - Consumers already maintain server-specific state, so global uniqueness buys nothing.
- Per-partition atomic counter + (partition, node) → unique → offsets fall out naturally as monotonic ints.

## Writes
- **Serial appends** to the active (last) segment.
- Segment rolls over to a new file at a configurable size (e.g. 1 GB) or time bound.
- Two flush parameters:
  - `M` — number of messages before forcing OS flush.
  - `S` — seconds before forcing flush.
- Durability bound: lose at most `M` messages or `S` seconds on crash.

## Reads
- Inputs: 64-bit logical offset + max chunk size `S` bytes.
- Returns iterator over messages contained in `S`-byte buffer.
- Process:
  1. Locate segment file via in-memory segment range (binary search).
  2. Translate global offset → file-local offset.
  3. Read from that file offset.
- ⚠️ Buffer may end on a partial message — size-delimiting makes detection trivial.
- If a single message exceeds `S`, the client doubles buffer and retries.
- Out-of-range offset → `OutOfRangeException`; client resets or fails per its policy.

## Reading "Latest"
- Log exposes the most recently written message so clients can subscribe at "now".
- Useful when consumers fall outside SLA retention and must skip ahead instead of failing.

## Result Format
```
MessageSetSend (fetch result)
  total length : 4
  error code   : 2
  message 1    : x
  ...
  message n    : x

MultiMessageSetSend (multiFetch)
  total length : 4
  error code   : 2
  messageSetSend 1
  ...
  messageSetSend n
```

## Deletes (Retention)
- Per segment, never per record (until compaction).
- Two policies:
  - **Time** — segment's largest record timestamp vs `retention.ms`.
  - **Size** — disabled by default; when on, drop oldest segments until partition fits `retention.bytes`.
- Both can be active simultaneously — eligible under either ⇒ deleted.
- Segment list uses copy-on-write so deletions don't block ongoing reads (binary search runs on an immutable snapshot).

## Recovery & Guarantees
- On startup, log recovery iterates the newest segment and validates each entry:
  - `size + offset < file length`
  - CRC-32C matches stored CRC.
- On corruption: truncate to last valid offset.
- 💡 Combined with replication, this means partial writes from crashes are safely discarded without corrupting the replica.

## Tags
[[Kafka]] [[Log Format]] [[Segments]] [[Index]] [[Brokers]]
