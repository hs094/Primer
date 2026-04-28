# 01 — Basic Operations

🔑 Day-2 admin via `kafka-*.sh` scripts: topics, configs, reassignment, leaders, consumer groups.

Source: https://kafka.apache.org/42/operations/basic-kafka-operations/

## Topics — `kafka-topics.sh`

| Action | Command |
|---|---|
| Create | `--create --topic foo --partitions 20 --replication-factor 3 --config x=y` |
| List | `--list` |
| Describe | `--describe --topic foo` |
| Alter (add partitions) | `--alter --topic foo --partitions 40` |
| Delete | `--delete --topic foo` |

All commands take `--bootstrap-server localhost:9092`.

- ⚠️ Adding partitions does **not** repartition existing data — breaks key→partition assumptions for keyed producers/consumers.
- ⚠️ Reducing partitions is **not supported**.
- ⚠️ Topic name max 249 chars (folder-name limit with partition IDs).
- Replication factor 2–3 typical to survive node loss.

## Per-Topic Config — `kafka-configs.sh`

```bash
# add/update
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics \
  --entity-name foo --alter --add-config retention.ms=86400000

# remove
kafka-configs.sh ... --alter --delete-config retention.ms

# describe
kafka-configs.sh ... --entity-type topics --entity-name foo --describe
```

- Same shape works for `users`, `clients`, `brokers` entity types.
- For [[Quotas]]: `--entity-type users --entity-name alice --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048'`. Use `--entity-default` for cluster-wide defaults.

## Partition Reassignment — `kafka-reassign-partitions.sh`

Three phases: **generate → execute → verify**.

```bash
# 1. generate plan
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json --broker-list "5,6" --generate

# 2. execute with throttle (bytes/sec)
kafka-reassign-partitions.sh --execute \
  --reassignment-json-file plan.json --throttle 50000000

# 3. verify (also clears throttles)
kafka-reassign-partitions.sh --verify --reassignment-json-file plan.json
```

- ⚠️ **Always run `--verify`** after completion — it removes throttles. Forgotten throttles silently cripple replication.
- 💡 Watch `ConsumerLag` metric on the moving replica; should monotonically drop.

## Graceful Shutdown

Set in broker config:

```
controlled.shutdown.enable=true
```

- Syncs logs to disk → no recovery scan on restart.
- Migrates leadership off the shutting broker first.
- Requires replication factor > 1 with at least one in-sync follower.

## Leader Balancing

```
auto.leader.rebalance.enable=true
```

- Periodically restores leadership to the **preferred replica** (first in the assignment list).
- Manual trigger: `kafka-leader-election.sh --election-type preferred --all-topic-partitions`.

## Consumer Position — `kafka-consumer-groups.sh`

```bash
# lag per partition
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group

# active members
... --describe --group my-group --members --verbose

# reset (group MUST be inactive)
... --reset-offsets --group my-group --topic t1 --to-latest --execute
```

Output columns: `CURRENT-OFFSET | LOG-END-OFFSET | LAG | CONSUMER-ID | HOST | CLIENT-ID`.

- ⚠️ Offset reset only works when **no consumers in the group are running**.

## Tags
[[Kafka]] [[Operations]] [[Quotas]]
