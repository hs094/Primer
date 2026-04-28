# 03 — Connector Development

🔑 Implement a `Connector` (config + task split) and a `Task` (`poll()` for source / `put()` for sink); the framework handles offsets, retries, and rebalancing.

Source: https://kafka.apache.org/42/kafka-connect/connector-development-guide/

## Class hierarchy

```
Connector (abstract)
├── SourceConnector  →  produces SourceTask instances
└── SinkConnector    →  produces SinkTask  instances

Task (abstract)
├── SourceTask
│     └── List<SourceRecord> poll()        // pull from external system
└── SinkTask
      ├── void put(Collection<SinkRecord>) // batch from Kafka
      └── void flush(...)                  // ensure durability before offset commit
```

- The `Connector` runs once on the leader, reads configuration, and decides how to slice work into `taskConfigs(int maxTasks)`.
- The `Task` runs on a worker per slice and does the I/O.

## Records and schema

| Type | What it carries |
| --- | --- |
| `ConnectRecord` (super) | topic, partition, key + keySchema, value + valueSchema, timestamp, headers |
| `SourceRecord` | + `sourcePartition` (Map) + `sourceOffset` (Map) |
| `SinkRecord` | + `kafkaOffset`, `kafkaPartition`, `timestampType` |
| `SchemaAndValue` | `(Schema schema, Object value)` — used by transforms / converters |
| `Schema` / `Struct` | Connect's runtime data model (typed, evolvable, converter-agnostic) |

## Source offset management
- `sourcePartition` = an opaque Map describing the *unit of work* (e.g. `{"db":"sales","table":"orders"}` or `{"file":"/var/log/app.log"}`).
- `sourceOffset` = an opaque Map describing the *position within that unit* (e.g. `{"lsn":12345}` or `{"position":4096}`).
- Connect commits offsets to `offset.storage.topic` (distributed) or local file (standalone).
- On startup, `SourceTask` calls `context.offsetStorageReader().offset(sourcePartition)` to resume.

```java
public List<SourceRecord> poll() {
    long pos = (Long) lastOffset.getOrDefault("position", 0L);
    List<SourceRecord> out = new ArrayList<>();
    for (var rec : readBatch(pos)) {
        out.add(new SourceRecord(
            Map.of("file", file),
            Map.of("position", rec.endPos()),
            "topic-name",
            Schema.STRING_SCHEMA, rec.line()));
    }
    return out;
}
```

## Delivery semantics
- **At-least-once is the default** for both source and sink.
- For sources, exactly-once is *opt-in* (worker config + connector validation, see [[Kafka Connect 04 - Administration]]).
- For sinks, "exactly-once" depends on the destination's idempotency; Connect can't transact across Kafka *and* an arbitrary system.

## Single Message Transforms ([[SMT]])
- Lightweight, per-record transforms applied in the pipeline:
  ```
  source → SMT → converter → broker     (source connector)
  broker → converter → SMT → sink       (sink connector)
  ```
- Configured on the **connector** config (not worker):

```properties
transforms=route,mask
transforms.route.type=org.apache.kafka.connect.transforms.RegexRouter
transforms.route.regex=(.*)
transforms.route.replacement=prod_$1
transforms.mask.type=org.apache.kafka.connect.transforms.MaskField$Value
transforms.mask.fields=ssn,email
```

- Built-in SMTs: `Cast`, `ExtractField`, `Filter`, `Flatten`, `HeaderFrom`, `HoistField`, `InsertField`, `MaskField`, `RegexRouter`, `ReplaceField`, `SetSchemaMetadata`, `TimestampConverter`, `TimestampRouter`, `ValueToKey`.
- Custom SMTs: implement `Transformation<R extends ConnectRecord<R>>`.

⚠️ SMTs run synchronously per record — keep them cheap. For heavy logic, use [[Kafka Streams]] downstream.

## Tags
[[Kafka]] [[Kafka Connect]] [[Connector]] [[SMT]]
