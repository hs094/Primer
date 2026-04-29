# 07 — Partitioning

🔑 Declarative partitioning splits one logical table into many physical children — the planner prunes irrelevant ones at query time.

Source: https://www.postgresql.org/docs/current/ddl-partitioning.html

## Strategies
| Strategy | Use |
|---|---|
| **RANGE** | Time-series (`created_at`), sequential ids |
| **LIST** | Discrete categories (region, tenant) |
| **HASH** | Even distribution when no natural key |

```sql
CREATE TABLE events (
  id bigint, at timestamptz NOT NULL, payload jsonb
) PARTITION BY RANGE (at);

CREATE TABLE events_2026_04
  PARTITION OF events
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE TABLE events_default PARTITION OF events DEFAULT;
```

## Partition Pruning
- Planner skips partitions that can't match `WHERE`.
- Works at **plan time** (constants) and **execution time** (params).
- 💡 Confirm via `EXPLAIN` — look for "Partitions pruned" or only the relevant child plans.

## Partition-wise Operations
- `enable_partitionwise_join = on` — joins between same-partitioned tables run per-partition.
- `enable_partitionwise_aggregate = on` — same for `GROUP BY`.

## Attach / Detach
```sql
-- Build child as a regular table, then bolt on
ALTER TABLE events ATTACH PARTITION events_2026_05
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Detach without locking out reads
ALTER TABLE events DETACH PARTITION events_2026_01 CONCURRENTLY;
```

## Default Partition
- Catches values not matching any partition.
- ⚠️ Adding a new partition that overlaps the default forces a full scan of the default to validate.

## Gotchas
- ⚠️ Indexes are per-partition; declare on parent → propagates.
- ⚠️ `UNIQUE` must include the partition key.
- ⚠️ FKs **referencing** a partitioned table work (14+); FKs **from** partitioned tables also work — but cross-partition uniqueness still requires the key.

## Tags
[[PostgreSQL]] [[Partitioning]] [[Performance]]
