# 07 — Tiered Storage

🔑 Hot tail on local disk, cold history offloaded to remote object store. Cheap long retention.

Source: https://kafka.apache.org/42/operations/tiered-storage/

## The Idea

- Tail reads (real-time consumers) hit page cache anyway.
- Old segments are rarely re-read — wasteful on premium SSD.
- Move closed segments to **S3 / GCS / HDFS** transparently.

## Cluster Config

```
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=<plugin FQCN>
remote.log.metadata.manager.class.name=<plugin FQCN>   # default uses internal topic
```

- ⚠️ **No built-in `RemoteStorageManager`** — supply a plugin (Aiven S3, Confluent S3/GCS, etc.).
- Default `RemoteLogMetadataManager` uses the `__remote_log_metadata` internal topic.

## Per-Topic Config

```
remote.storage.enable=true
local.retention.ms=86400000        # 1 day on local disk
local.retention.bytes=10737418240  # or by size
retention.ms=2592000000            # 30 days total (local + remote)
retention.bytes=...
```

| Knob | Controls |
|---|---|
| `local.retention.*` | When segments **move** to remote |
| `retention.*` | When segments are **deleted** entirely |

So `local.retention.ms < retention.ms` always.

## Plugin Surface

Implement two interfaces:

| Interface | Job |
|---|---|
| `RemoteStorageManager` | Read/write segment + index files in remote store |
| `RemoteLogMetadataManager` | Track which segments live where (offsets, leader epochs) |

## Limitations

- ⚠️ **Compacted topics not supported.**
- ⚠️ Disabling at cluster level requires disabling on **every topic first**.
- ⚠️ Pre-2.8 segments lacking producer snapshots can't be tiered.
- Remote fetches are slower — first cold read may stall a consumer.

## Try-It

🧪 `LocalTieredStorage` — bundled test plugin that tiers to a local directory. Use to dry-run config and consumer behavior without standing up S3.

## Tags
[[Kafka]] [[Tiered Storage]] [[Operations]]
