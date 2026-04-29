# 07 — Persistence

🔑 Two engines: RDB (point-in-time snapshots) and AOF (write-ahead log). Run both for safety; pick fsync policy by tolerance for data loss.

Source: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/

## RDB — Snapshots
- Periodic binary dump of the dataset (`dump.rdb`).
- Trigger via `save 60 1000` (every 60s if ≥1000 keys changed) or manual `BGSAVE`.
- Fork child does the I/O → main thread keeps serving.
- ✅ Compact, fast restart, ideal for backups (ship to S3).
- ❌ Lose everything since last snapshot on crash. ⚠️ `fork()` on a 100 GB instance can stall ms→s.

## AOF — Append-Only File
- Log every write command, replay on restart.
- Background rewrite compacts the log without downtime.
- ✅ Per-second durability; survives `FLUSHALL` (edit the file, drop the bad command).
- ❌ Larger on disk, slightly slower writes.

## fsync Policies
| `appendfsync` | Durability | Throughput | Worst-case loss |
|---|---|---|---|
| `always` | Strongest | Slowest | ~0 |
| `everysec` (default) | Strong | Fast | ≤ 1 second |
| `no` | Weakest (OS decides) | Fastest | tens of seconds |

## Hybrid (recommended for "data matters")
- Enable both. AOF rewrite preamble is an RDB blob → fast load + AOF tail for last-second writes.
- `aof-use-rdb-preamble yes` (default in modern versions).

## Tradeoff Matrix
| Goal | Config |
|---|---|
| Pure cache, OK to lose all | `save ""`, `appendonly no` |
| Backups, tolerate minutes loss | RDB only |
| Postgres-grade durability | RDB + AOF + `everysec` |
| Hardest durability | AOF + `always` (expect 10x latency hit) |

## 💡 Operational Notes
- Replicas should also persist; otherwise a master crash + auto-restart from empty replica wipes the dataset across the chain.
- `LASTSAVE`, `INFO persistence`, `BGREWRITEAOF` are your friends.
- ⚠️ Don't host RDB/AOF on the same disk as the OS swap — disk pressure during fork stalls everything.

## Tags
[[Redis]] [[Persistence]] [[RDB]] [[AOF]] [[Replication]]
