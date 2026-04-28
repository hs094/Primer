# 02 — Efficiency and Zero-Copy

🔑 Kafka pushes bytes from page cache to NIC via `sendfile`, skipping user-space — no copies, no serialization on the read path.

Source: https://kafka.apache.org/42/design/design/

## The Standard Path (4 copies, 2 syscalls)
1. Disk → kernel page cache (DMA).
2. Page cache → user buffer.
3. User buffer → kernel socket buffer.
4. Socket buffer → NIC (DMA).

## Kafka's Path with `sendfile`
- Page cache → NIC directly (Java `FileChannel.transferTo`).
- Eliminates 2 user-space copies and the user/kernel context switches.
- Multiple consumers fetching the same partition all serve from one shared page cache copy.

## TLS Gotcha
- ⚠️ When TLS/SSL is enabled, `sendfile` is bypassed — encryption is a user-space transformation, so bytes must pass through the JVM.
- Plaintext inter-broker / consumer traffic gets the full zero-copy benefit; encrypted traffic doesn't.

## Batching
- "Message set" abstraction: producers batch, network sends batch, broker appends batch, consumer fetches batch — same on-disk and on-wire layout.
- Producer thresholds: byte size or linger time (e.g. 64 KB or 10 ms).
- Amortizes RTT, syscalls, and TCP overhead.

## End-to-End Compression
| Codec | Notes |
|---|---|
| GZIP | Highest ratio, slowest |
| Snappy | Fast, decent ratio |
| LZ4 | Very fast |
| ZStandard | Best modern trade-off |
- Compression is applied to a record batch, not per-message — repeated field names, headers, and string values share dictionary state.
- Compressed batch is stored compressed on disk and fetched compressed; consumer decompresses.

## Tags
[[Kafka]] [[Zero-Copy]] [[sendfile]] [[Batching]] [[Compression]] [[Brokers]]
