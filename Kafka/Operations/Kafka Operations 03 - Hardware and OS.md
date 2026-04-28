# 03 — Hardware and OS

🔑 Kafka leans on the **OS page cache** and **disk throughput**. Don't starve either.

Source: https://kafka.apache.org/42/operations/hardware-and-os/

## Disks

- **More disks > faster disks.** Throughput is the bottleneck.
- Either RAID them or mount as separate `log.dirs` entries.
- ⚠️ **RAID-5 / RAID-6: avoid** — write penalty kills throughput. JBOD or RAID-10 if you must.
- Reference: 8× 7200 RPM SATA in upstream tests.

## Filesystem

| FS | Verdict |
|---|---|
| **XFS** | Preferred. ~160ms vs ext4's ~250ms+ in flush tests. Minimal tuning. |
| ext4 | Works but needs careful tuning. |

Mount with `noatime` to skip atime updates on reads.

## RAM / Page Cache

- Buffer ~30 seconds of write throughput in page cache.
- 1 GB/s write rate → ~30 GB cache target.
- 64 GB+ machines typical for production.
- Heap stays small; **leave the rest to page cache**.

## CPU

- Usually network/encryption-bound, not CPU-bound.
- Reference: dual quad-core Xeon class.
- SSL/SASL adds CPU cost — size up if encrypting.

## JVM

- **G1GC** is the default and recommended collector.
- Heap **4–8 GB** typical. Larger heaps reduce page cache room — counterproductive.
- Don't oversize heap "just in case" — the kernel cache is doing the heavy lifting.

## OS Tuning

| Setting | Recommendation |
|---|---|
| File descriptors | ≥ 100,000 per broker process |
| `vm.max_map_count` | Raise — each segment uses 2 maps; default 65,535 crashes high-partition brokers |
| `vm.swappiness` | `1` (avoid swap, don't disable entirely) |
| `vm.dirty_ratio` | Tune to control flush burstiness |
| Socket buffers | Increase for cross-DC links |

⚠️ Hitting `vm.max_map_count` manifests as `OutOfMemoryError: Map failed` — symptom is misleading.

## Tags
[[Kafka]] [[Operations]]
