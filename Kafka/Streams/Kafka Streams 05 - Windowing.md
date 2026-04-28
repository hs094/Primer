# 05 — Windowing

🔑 Windows bucket records by event-time so aggregates have closed boundaries; grace controls late data.

Source: https://kafka.apache.org/42/streams/developer-guide/dsl-api/#windowing

## Window types

| Type | Shape | Defined by | When the window emits |
| --- | --- | --- | --- |
| **Tumbling** | Fixed, non-overlapping | `size` | Aligned to epoch; one window per `size` |
| **Hopping** | Fixed, overlapping | `size` + `advance (< size)` | Multiple windows cover each record |
| **Sliding** | Fixed `size`, dynamic boundaries | `size` + `grace` | Defined by record time differences (≤ size apart) |
| **Session** | Variable, activity-driven | `inactivity gap` | Closes after `gap` of silence; merges on new record within gap |

```java
// tumbling 1 min
.windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(10)))
// hopping 5 min size, 1 min advance
.windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofSeconds(30))
            .advanceBy(Duration.ofMinutes(1)))
// session, 30 s inactivity
.windowedBy(SessionWindows.ofInactivityGapAndGrace(Duration.ofSeconds(30), Duration.ofSeconds(10)))
```

## Grace period (allowed lateness)
- Defines how long after a window closes the engine still accepts late records into it.
- Records arriving past `windowEnd + grace` are dropped (counted in `late-record-drop` metrics).
- In Streams 4.x, grace is **mandatory** and must be supplied via `…AndGrace(...)` factory.
- ⚠️ Larger grace ⇒ more state retained ⇒ more memory.

## Final results — `suppress`
- By default, windowed aggregates emit a result *every time* a record updates them.
- For "fire once when the window closes" semantics, use:

```java
.suppress(Suppressed.untilWindowCloses(Suppressed.BufferConfig.unbounded()))
```

- ⚠️ `untilWindowCloses` requires a buffer — bound it (`maxBytes`/`maxRecords`) in production unless you trust the cardinality.

## Time semantics

| Notion | What it is | How to set |
| --- | --- | --- |
| **Event-time** (default) | Embedded record timestamp (producer or extractor) | `TimestampExtractor` (default reads `ProducerRecord.timestamp()`) |
| **Ingestion-time** | Time when broker stored the record | Topic config `message.timestamp.type=LogAppendTime` |
| **Processing-time** | Wall clock at the processor | `WallclockTimestampExtractor` |

💡 Stick with event-time for correctness; only use processing-time for purely operational metrics.

## Tags
[[Kafka]] [[Kafka Streams]] [[KStream]] [[KTable]]
