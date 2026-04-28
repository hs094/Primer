# 06 — Joins

🔑 Pick the join shape by the operands' types; co-partitioning is required everywhere except GlobalKTable.

Source: https://kafka.apache.org/42/streams/developer-guide/dsl-api/#joining

## Join matrix

| Left | Right | Windowed? | Inner / Left / Outer | Notes |
| --- | --- | --- | --- | --- |
| [[KStream]] | KStream | ✅ required (`JoinWindows`) | all three | Symmetric; both sides buffered for the window |
| KStream | [[KTable]] | ❌ | inner, left | Stream side drives lookups; table is the dimension |
| KStream | GlobalKTable | ❌ | inner, left | Lookup by **arbitrary** field via `KeyValueMapper` |
| KTable | KTable (primary-key) | ❌ | all three | Both sides materialized; result is a KTable |
| KTable | KTable (foreign-key) | ❌ | inner, left | Joins on a non-PK column; uses internal subscription topic |

## Co-partitioning requirement

| Pair | Co-partitioning required? |
| --- | --- |
| KStream–KStream | ✅ |
| KStream–KTable | ✅ |
| KTable–KTable (PK) | ✅ |
| KTable–KTable (FK) | ❌ (Streams handles repartitioning internally) |
| KStream–GlobalKTable | ❌ |

**Co-partitioning** = (a) same number of partitions on both topics, (b) same partitioner, (c) same key type/serdes. If violated, Streams will refuse to start (or you must call `repartition()` / `through()` to fix).

⚠️ A `selectKey` / `map` that changes the key invalidates co-partitioning — Streams adds an automatic repartition topic before the stateful op.

## Windowed stream-stream join

```java
left.join(
    right,
    (l, r) -> l + "|" + r,
    JoinWindows.ofTimeDifferenceAndGrace(Duration.ofMinutes(5), Duration.ofSeconds(30)),
    StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
);
```

- Both sides are stored in window stores for `size + grace`.
- For `outer` / `left`, "missing" matches are emitted when the window closes.

## KStream–GlobalKTable

```java
stream.join(
    globalUserTable,
    (orderKey, order) -> order.userId(),   // KeyValueMapper picks join key
    (order, user) -> enrich(order, user)
);
```

- No repartition, no co-partitioning — global table is fully replicated.
- 💡 Use for small reference data (countries, currencies, feature flags); large globals waste RAM/disk on every instance.

## Tags
[[Kafka]] [[Kafka Streams]] [[KStream]] [[KTable]]
