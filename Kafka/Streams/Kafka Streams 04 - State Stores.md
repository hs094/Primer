# 04 — State Stores

🔑 Local RocksDB (or in-memory) stores back stateful operators; a Kafka changelog topic makes them fault-tolerant.

Source: https://kafka.apache.org/42/streams/architecture/ (state)

## What lives in a store
- Aggregates (`count`, `reduce`, `aggregate` results).
- Windowed aggregates (key + window → value).
- Materialized [[KTable]]s.
- Custom stores via the Processor API (`addStateStore`).

## Backing engines

| Type | Class | Notes |
| --- | --- | --- |
| Persistent (default) | RocksDB | Survives restarts, spills to disk. `state.dir` controls location. |
| In-memory | `Stores.inMemoryKeyValueStore` | Bounded by JVM heap; faster but rebuilt fully on restart. |
| Versioned | `Stores.persistentVersionedKeyValueStore` | Time-travel reads for temporal joins. |
| Window / Session | RocksDB or in-memory variants | Used implicitly by windowed ops. |

## Fault tolerance — the changelog topic
- Every store auto-creates a compacted topic named `<application.id>-<store-name>-changelog`.
- Every write to the store is also written to the changelog (in the same transaction under EOS-v2).
- On task failover, the new owner restores the store by replaying the changelog from the start.
- ⚠️ Restore time scales with changelog size — keep stores reasonably small or use standby replicas.

## Standby replicas
- `num.standby.replicas` (default `0`): keep N hot copies of each task's state on other instances.
- A standby continuously consumes the changelog and warms RocksDB.
- On failover, a standby takes over with **near-zero** restore time vs cold rebuild.
- Tradeoff: `N` extra disk + memory + network per standby.

```properties
num.standby.replicas=1
acceptable.recovery.lag=10000
```

## Interactive Queries
- Store contents are queryable from within the app via `KafkaStreams.store(...)`.
- Use `StreamsMetadata` / `KafkaStreams.metadataForKey(...)` to discover which instance owns a key (state is partitioned, so any given key lives on exactly one active task).
- 🧪 Try-it: expose `/lookup/{key}` REST endpoint that proxies to the right instance using the metadata API.

💡 For external read traffic, pair an interactive-query endpoint with a small reverse-proxy/discovery layer rather than dumping state to a separate DB.

## Tags
[[Kafka]] [[Kafka Streams]] [[State Store]] [[KTable]]
