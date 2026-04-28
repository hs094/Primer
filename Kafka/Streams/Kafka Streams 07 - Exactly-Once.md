# 07 — Exactly-Once Processing

🔑 `processing.guarantee=exactly_once_v2` wraps consume-process-produce *plus* state updates in a single Kafka transaction.

Source: https://kafka.apache.org/42/streams/core-concepts/#processing-guarantees

## The setting

```properties
processing.guarantee=exactly_once_v2     # the modern option
# or
processing.guarantee=at_least_once       # default; faster, but duplicates possible on failure
```

- `exactly_once_v2` (a.k.a. EOS v2) is the supported exactly-once mode in Kafka 4.x.
- Older `exactly_once` and `exactly_once_beta` are deprecated/removed.
- `at_least_once` is the *default* for safety/perf; flip to `exactly_once_v2` when correctness > throughput.

## What "exactly-once" actually covers
A Streams task does three things per input record:
1. Read from input topic(s).
2. Update local [[State Store]] (and its changelog).
3. Produce to output topic(s) and commit the input offset.

Under EOS-v2, **all three are atomic**: either everything is visible / committed or none of it. Implemented by a single Kafka producer transaction per task per commit interval.

```
[input offsets]  +  [changelog writes]  +  [output records]   ─►  one transaction
```

## Requirements
- **Brokers ≥ 2.5** (transactions + idempotent producer; 4.x brokers more than satisfy this).
- `transaction.state.log.replication.factor` and `transaction.state.log.min.isr` configured on the broker side.
- Producer is implicitly idempotent and transactional — Streams configures this for you.
- Read isolation: downstream consumers must use `isolation.level=read_committed` to actually skip aborted records.

## Costs / knobs
- `commit.interval.ms` defaults to **100 ms** under EOS-v2 (vs 30 s under at-least-once) ⇒ more, smaller transactions ⇒ lower throughput.
- More heap / IO from frequent transactional commits.
- Failover is fast: aborted in-flight transactions get rolled back at the broker.

⚠️ Side effects in your processors (HTTP calls, external DB writes) are **not** transactional. EOS only covers Kafka — anything outside Kafka must be idempotent on its own.

💡 If you don't need EOS for an entire topology, run two separate apps with different `application.id` and only enable EOS on the one that needs it.

## Tags
[[Kafka]] [[Kafka Streams]] [[Exactly-Once]] [[State Store]]
