# 08 — Replication

🔑 Async leader-follower: master streams writes to replicas, replicas serve reads. Partial resync via replication backlog avoids full RDB transfers on brief disconnects.

Source: https://redis.io/docs/latest/operate/oss_and_stack/management/replication/

## How It Works
1. Replica connects, sends `PSYNC <replid> <offset>`.
2. Master either:
   - Streams the missing offsets from the **replication backlog** (partial resync), or
   - Snapshots an RDB, ships it, then streams subsequent writes (full resync).
3. After sync, master continuously sends each write command in arrival order.

## Async by Default
- Master replies `OK` to the client **before** the write reaches replicas.
- Crash before propagation → that write is lost. This is the price of low latency.
- `WAIT n ms` blocks until `n` replicas ack — synchronous-ish writes when you need them.

## Configuration
```conf
# replica
replicaof 10.0.0.1 6379
masterauth s3cret
replica-read-only yes

# master
repl-backlog-size 64mb        # bigger backlog → tolerates longer disconnects
repl-diskless-sync yes        # stream RDB over the socket, skip disk
min-replicas-to-write 1       # refuse writes if no replica is in sync
min-replicas-max-lag 10
```

## Replication ID + Offset
- Master has a `replid` (random) and an ever-growing byte offset.
- `replid` changes on promotion → forces a full resync from the new master.
- `INFO replication` shows `master_replid`, `master_repl_offset`, per-replica lag.

## Failover
- Manual: `REPLICAOF NO ONE` on a chosen replica → it becomes master.
- Automatic: run [[Sentinel]] or [[Redis Cluster]].

## Topologies
| Shape | Note |
|---|---|
| 1 master + N replicas | Most common; reads scale, writes don't |
| Cascading (master → replica → replica) | Reduce master fan-out, +1 hop of lag |
| Replica-only `replica-read-only no` | Ad-hoc analytics replica; ⚠️ writes here diverge and break replication |

## ⚠️ Gotchas
- Don't run a master with persistence off **and** auto-restart — replicas resync from an empty master and lose everything.
- Replica lag spikes = master is faster than network or replica disk; check `repl_backlog_size` and `repl-diskless-sync-delay`.

## Tags
[[Redis]] [[Replication]] [[Sentinel]] [[Redis Cluster]] [[Persistence]]
