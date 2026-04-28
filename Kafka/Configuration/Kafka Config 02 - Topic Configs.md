# 02 — Topic Configs

🔑 Per-topic overrides of broker defaults. Set at create time (`--config k=v`) or later via `kafka-configs.sh --entity-type topics`. Topic-level wins over broker-level.

Source: https://kafka.apache.org/42/configuration/topic-configs/

## Retention & Cleanup

| key | type | default | meaning |
| --- | --- | --- | --- |
| `cleanup.policy` | list | `delete` | `delete`, `compact`, or `delete,compact`. `compact` = key-based log compaction (latest value per key). |
| `retention.ms` | long | 604800000 (7d) | Time-based retention before discarding old segments. `-1` = forever. |
| `retention.bytes` | long | -1 | Per-partition size cap. `-1` = unlimited. |
| `segment.bytes` | int | 1 GiB | Active segment size before roll. Smaller = faster expiry, more files. |
| `segment.ms` | long | 604800000 (7d) | Force segment roll after this time even if not full. |
| `delete.retention.ms` | long | 86400000 (1d) | How long compaction keeps tombstones (null-value deletes) for compacted topics. |
| `min.cleanable.dirty.ratio` | double | 0.5 | Compactor triggers when dirty/total ratio exceeds this. Lower = more aggressive cleaning. |
| `min.compaction.lag.ms` | long | 0 | Minimum age before a record becomes eligible for compaction. |
| `max.compaction.lag.ms` | long | Long.MAX | Hard upper bound on age before compaction must run — useful for GDPR-style "must-be-cleaned" SLAs. |

## Durability

| key | type | default | meaning |
| --- | --- | --- | --- |
| `min.insync.replicas` | int | 1 | Min ISR for `acks=all` writes. Set to 2 with RF=3 for safety. |
| `unclean.leader.election.enable` | boolean | false | Per-topic override of broker config — keep false unless availability > durability. |

## Messages

| key | type | default | meaning |
| --- | --- | --- | --- |
| `max.message.bytes` | int | 1048588 (~1 MiB) | Max record-batch size after compression. Producer's `max.request.size` must match. |
| `compression.type` | string | `producer` | `producer` (keep producer's codec), `gzip`, `snappy`, `lz4`, `zstd`, `uncompressed`. |
| `message.timestamp.type` | string | `CreateTime` | `CreateTime` = producer's timestamp; `LogAppendTime` = broker overwrites on append. |
| `message.timestamp.before.max.ms` | long | Long.MAX | Reject records whose timestamp is older than `now - this`. |
| `message.timestamp.after.max.ms` | long | 3600000 (1h) | Reject records whose timestamp is more than this in the future. |

## Tiered Storage (per-topic)

| key | type | default | meaning |
| --- | --- | --- | --- |
| `remote.storage.enable` | boolean | false | Enable tiering for this topic. Cluster must have `remote.log.storage.system.enable=true`. |
| `local.retention.ms` | long | -2 | How long segments stay on local disk after upload. `-2` = inherit `retention.ms`. |
| `local.retention.bytes` | long | -2 | Local-disk size cap per partition. `-2` = inherit `retention.bytes`. |

⚠️ `cleanup.policy=compact` requires keys on every record — null-key records are dropped silently by compaction.

💡 For event-sourcing / CDC keys: `cleanup.policy=compact,delete` + `retention.ms` gives "keep latest per key, but eventually drop everything" semantics.

🧪 Try it: `kafka-configs.sh --bootstrap-server :9092 --entity-type topics --entity-name orders --alter --add-config retention.ms=259200000`.

## Tags
[[Kafka]] [[Topics]] [[Tiered Storage]]
