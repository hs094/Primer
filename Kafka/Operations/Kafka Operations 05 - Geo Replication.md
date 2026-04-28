# 05 — Geo Replication (MirrorMaker 2)

🔑 [[MirrorMaker]] 2 mirrors topics, configs, [[ACL]]s, and consumer offsets across clusters via Kafka Connect.

Source: https://kafka.apache.org/42/operations/geo-replication-cross-cluster-data-mirroring/

## What MM2 Replicates

- Topic data (records).
- Topic configs.
- ACLs.
- Consumer group offsets (so apps can fail over and resume).
- Auto-detects new topics + partition expansions.

## Topology Modes

| Mode | Flow | Use Case |
|---|---|---|
| **Active/Passive** | A → B | DR / read replicas |
| **Active/Active** | A ↔ B | Multi-region writes |
| **Hub-and-Spoke** | A,B,C → K (aggregate) or K → A,B,C (fan-out) | Central analytics |
| **Multi-cluster mesh** | Arbitrary edges | Federated DCs |

Flows are declared as `source->target` properties.

## Loop Prevention via Naming

Default replicated topic name: **`{source}.{topic}`**

```
us-west.orders   <- replicated from us-west to us-east
```

- Prefix encodes provenance → records from one cluster never overwrite the same partition on the destination.
- Configurable via `replication.policy.separator` and `replication.policy.class`.

## Replication Policies

| Class | Behavior |
|---|---|
| `DefaultReplicationPolicy` | Adds source-cluster prefix. Safe for active/active. |
| `IdentityReplicationPolicy` | Same name on both sides. ⚠️ Only safe for active/passive — loops in active/active. |

## Built On Connect

- MM2 = a set of Kafka Connect **source connectors** (`MirrorSourceConnector`, `MirrorCheckpointConnector`, `MirrorHeartbeatConnector`).
- Run on a dedicated Connect cluster or via `connect-mirror-maker.sh`.
- Per-flow override of producer/consumer/admin client configs in the props file.

## Operational Gotchas

- ⚠️ Offset translation is approximate — same-named consumer group on remote cluster sees translated offsets, not source offsets.
- ⚠️ ACL replication needs the source `User:` principal to make sense in the destination's auth realm.
- 💡 Use `MirrorHeartbeatConnector` to detect replication staleness via end-to-end heartbeat lag.

## Tags
[[Kafka]] [[MirrorMaker]] [[Operations]]
