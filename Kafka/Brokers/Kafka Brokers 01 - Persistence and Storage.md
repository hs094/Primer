# 01 — Persistence and Storage

🔑 Kafka writes append-only to disk sequentially and leans on the OS page cache instead of an in-process buffer — disks aren't slow if you stop seeking.

Source: https://kafka.apache.org/42/design/design/

## Sequential vs Random Disk
| Access pattern | Modern SATA throughput |
|---|---|
| Sequential | ~600 MB/s |
| Random | ~100 kB/s |
- ~6000x difference. JBOD with 6×7200rpm SATA RAID-5 hits this.
- Linear reads/writes are the most predictable disk pattern and the OS aggressively prefetches them.

## Page Cache, Not Heap Cache
- Writes go through the filesystem to the OS page cache — Kafka does not maintain a parallel in-process cache.
- Kernel already keeps a buffer; duplicating it doubles memory use.
- Bytes on heap are ~2x the size of compact on-disk format (object header + GC overhead).
- 💡 32 GB box typically yields 28-30 GB of usable cache.

## GC and Restart Properties
- No big in-process cache → minimal heap pressure, GC stays cheap as data volume grows.
- Cache survives broker restarts (lives in kernel) — cold-start penalty is small.
- Filesystem flushes are coalesced by the kernel; Kafka doesn't need to fsync per write to be durable (replication handles durability).

## Constant-Time Operations
- Append: O(1).
- Read by offset: O(1) via segment + offset index lookup.
- ⚠️ Performance is decoupled from data size — a 50 GB partition reads as fast as a 50 MB one.

## Tags
[[Kafka]] [[Page Cache]] [[Sequential IO]] [[Brokers]]
