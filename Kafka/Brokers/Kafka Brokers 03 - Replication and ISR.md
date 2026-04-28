# 03 — Replication and ISR

🔑 Kafka uses a primary-backup ISR scheme — a write is "committed" once every in-sync replica has applied it; f+1 replicas tolerate f failures.

Source: https://kafka.apache.org/42/design/design/

## Topology
- Each partition has 1 leader + 0..N followers.
- Leader handles all reads and writes for the partition; followers fetch like consumers and append to their own log.
- Replication unit is the partition (not the broker).

## ISR — In-Sync Replica Set
- Subset of replicas that are caught up to the leader.
- Leader maintains and persists ISR membership.
- A replica is in ISR iff:
  1. It heartbeats with the controller within `broker.session.timeout.ms`.
  2. Its log lag from leader stays under `replica.lag.time.max.ms`.
- Falling behind on either drops it from ISR.

## Commit Semantics
- A message is **committed** once all replicas in the current ISR have appended it.
- Only committed messages are visible to consumers.
- 💡 Producers can choose whether to wait for commit via `acks`.

## Producer `acks`
| acks | Wait for | Durability |
|---|---|---|
| `0` | Nothing | Fire-and-forget; data loss possible |
| `1` | Leader write | Loses data if leader fails before replication |
| `all` | Full ISR commit | Safe as long as ≥1 ISR replica survives |

## `min.insync.replicas`
- Topic/broker config: minimum ISR size required for `acks=all` writes.
- Below threshold → producer gets `NotEnoughReplicas`.
- Typical pattern: `replication.factor=3`, `min.insync.replicas=2`. Tolerates 1 failure while still accepting writes.

## Quorum Math: f+1 vs 2f+1
| Scheme | Replicas to tolerate f failures |
|---|---|
| Majority vote (Raft, ZAB) | 2f+1 |
| Kafka ISR | f+1 |
- ISR's edge: tolerate f failures with f+1 replicas because every commit waits for *all* live ISR members, not a majority.
- Trade-off: write latency is bounded by the slowest in-sync follower, not the median.

## Unclean Leader Election
- If all ISR members die, choices are:
  1. Wait for an ISR replica to come back (consistency).
  2. Elect any surviving replica leader, possibly losing committed messages (availability).
- `unclean.leader.election.enable` — defaults to `false` since 0.11.0.0 (consistency over availability).
- ⚠️ Flipping to `true` is an explicit availability choice; expect data loss.

## Liveness
- Heartbeat: broker → controller (`broker.session.timeout.ms`).
- Replication lag: follower fetch position vs leader log end (`replica.lag.time.max.ms`).
- Both must be healthy for ISR membership.

## Tags
[[Kafka]] [[Replication]] [[ISR]] [[Leader Election]] [[Brokers]]
