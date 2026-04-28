# 09 — Distribution

🔑 Consumer groups are coordinated by a per-group broker (the **group coordinator**); offsets live in a compacted internal topic; cluster metadata in 4.x is managed by [[KRaft]] (no ZooKeeper).

Source: https://kafka.apache.org/42/implementation/distribution/

## Consumer Offset Tracking
- Each consumer tracks the max offset consumed per partition.
- Commits let consumers resume after restart.
- Offsets are stored in a designated broker per group — the **group coordinator**.
- All commits and fetches for that group route to its coordinator.

## Discovering the Coordinator
- Group → coordinator mapping is by hash of the group name onto partitions of `__consumer_offsets`.
- Client issues `FindCoordinatorRequest` to any broker; response carries coordinator host/port.
- ⚠️ If the coordinator partition's leader moves (rebalance, broker death), client must rediscover.

## `__consumer_offsets`
- Internal compacted topic — only the latest offset per (group, topic, partition) tuple matters.
- On `OffsetCommitRequest`:
  1. Coordinator appends the commit to its `__consumer_offsets` partition.
  2. Replies success only after **all replicas** of that partition have the record.
  3. Replication timeout → commit fails, consumer retries with backoff.
- Brokers periodically compact this topic to bound size.
- Coordinator caches offsets in memory for fast `OffsetFetchRequest` response.
- 💡 Cold-start: when a broker becomes coordinator for a group (it just gained leadership of the right `__consumer_offsets` partition), it loads that partition into the cache before serving fetches.

## Auto vs Manual Commit
- `enable.auto.commit=true` — periodic commits in the background; simpler, weaker guarantees.
- Manual `commitSync` / `commitAsync` — explicit, gives at-least-once or transactional integration.

## KRaft (4.x)
- Kafka 4.x is **KRaft-only** — ZooKeeper is gone.
- A small set of **controller nodes** runs a Raft replicated log holding all cluster metadata: topics, partitions, configs, ACLs, producer IDs, etc.
- Brokers replay the metadata log to stay in sync; one elected **active controller** writes new metadata records.
- Failover is fast — followers already have the full log, election promotes one to active.

## Partition Assignment
- Topic → N partitions, each with a replication factor R.
- Replicas spread across brokers (and racks if rack-awareness enabled) to survive single-node and rack failures.
- Leader for a partition is elected from its [[ISR]] by the active KRaft controller.
- On broker death: controller observes session timeout, picks new leaders from ISRs, propagates updated metadata.

## Tags
[[Kafka]] [[KRaft]] [[Group Coordinator]] [[Consumers]] [[Distribution]] [[Brokers]]
