# 02 — KRaft

🔑 Raft-based controller quorum replaces [[ZooKeeper]] entirely in Kafka 4.x. No ZK at all.

Source: https://kafka.apache.org/42/operations/kraft/

## What Changed

- Cluster metadata (topics, partitions, ACLs, configs) lives in an internal `__cluster_metadata` log.
- Controllers replicate metadata via Raft consensus.
- ⚠️ **Kafka 4.x is KRaft-only** — no ZK mode, no in-place ZK→KRaft migration. Must migrate on a 3.x bridge release first (last bridge: **3.9**).

## Process Roles

`process.roles` determines node function:

| Value | Meaning |
|---|---|
| `broker` | Serves clients, hosts partitions |
| `controller` | Quorum member, manages metadata |
| `broker,controller` | Combined — dev/small only |

- 💡 Production: dedicated controllers + dedicated brokers. Combined mode loses isolation and blocks independent scaling.

## Controller Quorum

- Typical size: **3 or 5** controllers.
- Tolerates `floor((N-1)/2)` failures: 3→1, 5→2.
- Majority must be alive for availability.

## Bootstrap — `kafka-storage.sh`

```bash
# generate cluster ID once
CLUSTER_ID=$(kafka-storage.sh random-uuid)

# format storage on every node
kafka-storage.sh format -t $CLUSTER_ID -c config/server.properties --standalone
```

Format modes:
- `--standalone` — single initial controller.
- `--initial-controllers` — multi-controller bootstrap.
- `--no-initial-controllers` — for brokers joining an existing cluster.

⚠️ Same `CLUSTER_ID` on every node — mismatched IDs refuse to join.

## Quorum Membership

| Style | Config | Notes |
|---|---|---|
| **Dynamic** (KRaft v1+, 4.1+) | `controller.quorum.bootstrap.servers` | Add/remove controllers at runtime |
| **Static** (legacy) | `controller.quorum.voters=1@host:9093,2@...` | Membership baked in |

## Listeners

- Controllers use a separate listener — see `controller.listener.names`.
- Cannot reuse the inter-broker listener name on controllers.

## Migration

- ⚠️ Kafka 4.x: **no ZK migration path**. If still on ZK, upgrade through 3.9 → migrate → then 4.x.

## Tags
[[Kafka]] [[KRaft]] [[ZooKeeper]] [[Operations]]
