# 02 — Core Concepts

🔑 Streams and tables are two views of the same log; a topology is a DAG of nodes; a task is one slice of that DAG bound to one input partition.

Source: https://kafka.apache.org/42/streams/core-concepts

## Stream–table duality
- **Stream**: unbounded, append-only sequence of immutable facts (`(key, value, ts)` tuples).
- **Table**: latest value per key — a snapshot.
- They convert losslessly: replay a stream's changelog ⇒ table; iterate table change events ⇒ stream.
- This is what makes state fault-tolerant: every store has a Kafka changelog topic.

## Three abstractions

| Abstraction | Semantics | Storage | Use when |
| --- | --- | --- | --- |
| [[KStream]] | Each record = INSERT (independent event) | None on its own | Logs, clicks, transactions, anything append-only |
| [[KTable]] | Each record = UPSERT/DELETE (tombstone = `null` value) | Local, partitioned + changelog | Latest-value-per-key state, dimension data |
| **GlobalKTable** | Same as KTable but **fully replicated to every instance** | Local, full copy + changelog | Small reference data; star-schema lookups without co-partitioning |

⚠️ GlobalKTable is bootstrapped on startup and uses `null` keys for the join lookup — it lets you join a KStream on a *non-key* attribute.

## Processor topology
- Directed acyclic graph of three node kinds:
  - **Source** — reads a topic.
  - **Processor** — transforms / aggregates / joins.
  - **Sink** — writes a topic.
- Built declaratively via the DSL (`StreamsBuilder`) or imperatively via the Processor API.
- A single application may have multiple sub-topologies (split when there is no in-process linkage between them).

## Stream task
- **Stream task = one copy of the topology bound to one input partition** (per source topic).
- Tasks are the unit of parallelism *and* the unit of failure recovery.
- `# tasks = max(input partitions across source topics)`. Adding instances redistributes tasks; you cannot exceed task count by adding instances.

```
topic A (3 parts)  ─┐
topic B (3 parts)  ─┼─►  3 tasks  ─►  spread across N instances (N ≤ 3)
```

## Time
- Each record carries a timestamp (event-time by default, set by the producer or a `TimestampExtractor`).
- Streams advances **stream time** = max event-time observed per task; windows/joins use this, not wall-clock.

## Tags
[[Kafka]] [[Kafka Streams]] [[KStream]] [[KTable]] [[State Store]]
