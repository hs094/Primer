# 04 — ClickHouse Distributed

🔑 ClickHouse separates two axes: **sharding** (partition data across nodes) and **replication** (copy data for HA). They compose.

Source: https://clickhouse.com/docs/engines/table-engines/special/distributed

## Sharding — `Distributed` engine
A `Distributed` table holds **no data** — it's a routing layer over local tables on each shard.
```sql
-- on every node, the local storage:
CREATE TABLE events_local ON CLUSTER my_cluster (...) ENGINE = MergeTree ORDER BY ...;

-- on every node, the fan-out view:
CREATE TABLE events ON CLUSTER my_cluster AS events_local
ENGINE = Distributed(my_cluster, default, events_local, cityHash64(user_id));
```
- **Sharding key** (`cityHash64(user_id)`) → assigns row to a shard.
- Reads parallelize across all shards; coordinator merges results.
- Writes can hit the Distributed table (async forward) or directly to local (faster, your batching).

## Replication — `ReplicatedMergeTree`
Replicas inside a shard stay in sync via [[ClickHouseKeeper]] (or ZooKeeper):
```sql
CREATE TABLE events_local ...
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',  -- ZK path
    '{replica}'                            -- replica name
)
ORDER BY ...;
```
- Insert lands on one replica → log entry in Keeper → other replicas pull the part.
- Async multi-master: any replica accepts writes.

## ClickHouse Keeper
🔑 Built-in C++ rewrite of ZooKeeper, **wire-compatible** with ZK clients. Runs as a sidecar or embedded. Use it instead of JVM ZooKeeper for new clusters.

## Topology
```
                ┌─ replica A ─┐    ┌─ replica A ─┐
shard 1 ────────┤             │    │             ├──── shard 2
                └─ replica B ─┘    └─ replica B ─┘
                       │                  │
                       └─── ClickHouse Keeper ───┘
```

⚠️ `internal_replication = true` in cluster config → Distributed writes one replica per shard, lets ReplicatedMergeTree replicate. Setting `false` causes double-writes.

💡 Shard for **scale** (data volume / parallelism), replicate for **HA** (read scale-out + failover).

## Tags
[[Clickhouse]] [[Sharding]] [[Replication]] [[Distributed]] [[ClickHouseKeeper]]
