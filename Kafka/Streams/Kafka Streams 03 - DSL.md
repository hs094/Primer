# 03 — Streams DSL

🔑 `StreamsBuilder` + fluent operators on `KStream` / `KTable` give you a high-level declarative pipeline.

Source: https://kafka.apache.org/42/streams/developer-guide/dsl-api/

## Entry point
- `StreamsBuilder builder = new StreamsBuilder();`
- `builder.stream(topic)` → [[KStream]] (every record = independent event)
- `builder.table(topic)` → [[KTable]] (every record = upsert; `null` value = delete)
- `builder.globalTable(topic)` → GlobalKTable (full local replica)
- Build and start: `new KafkaStreams(builder.build(), props).start();`

## Stateless operators (no state store needed)

| Operator | Effect |
| --- | --- |
| `filter` / `filterNot` | Keep / drop on predicate |
| `map` / `mapValues` | Transform record (`mapValues` keeps key, no repartition) |
| `flatMap` / `flatMapValues` | 0..N output records per input |
| `branch` / `split().branch(...)` | Route into multiple streams |
| `selectKey` | Re-key (forces repartition before stateful ops) |
| `merge` | Union two streams |
| `peek` | Side-effect, no transform |
| `to(topic)` / `through(topic)` | Sink / sink-and-rebound |

## Stateful operators (need a [[State Store]])

| Operator | Notes |
| --- | --- |
| `groupByKey` / `groupBy` | Prep step; `groupBy` repartitions, `groupByKey` does not |
| `count()` | Long count per key |
| `reduce(adder)` | Combine values, same type |
| `aggregate(init, adder)` | Custom accumulator type |
| `windowedBy(...)` | Tumbling / hopping / sliding / session windows |
| `join` / `leftJoin` / `outerJoin` | KStream-KStream (windowed), KStream-KTable, KTable-KTable, KStream-GlobalKTable |
| `suppress(Suppressed.untilWindowCloses(...))` | Emit only final window result |

## Minimal WordCount

```java
Properties p = new Properties();
p.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount");
p.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

StreamsBuilder b = new StreamsBuilder();
b.<String, String>stream("text-input")
 .flatMapValues(v -> Arrays.asList(v.toLowerCase().split("\\W+")))
 .groupBy((k, word) -> word)
 .count()
 .toStream()
 .to("word-counts", Produced.with(Serdes.String(), Serdes.Long()));

new KafkaStreams(b.build(), p).start();
```

💡 `mapValues` is preferred over `map` whenever you only touch the value — it skips the implicit repartition that `selectKey`/`map` triggers.

## Tags
[[Kafka]] [[Kafka Streams]] [[KStream]] [[KTable]]
