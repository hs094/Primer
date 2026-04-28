# 03 — Quickstart

🔑 Format storage with a cluster UUID, start the broker, and produce/consume from CLI — no ZooKeeper, KRaft is the only mode in 4.x.

Source: https://kafka.apache.org/42/getting-started/quickstart/

## Prereqs
- Java 17+
- Default broker listener: `localhost:9092`

## 1. Download
```bash
tar -xzf kafka_2.13-4.2.0.tgz
cd kafka_2.13-4.2.0
```

## 2. Bootstrap KRaft
⚠️ ZooKeeper was removed in 4.0. KRaft (controller embedded in broker) is the only path.

```bash
# Generate cluster UUID
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

# Format log dirs (one-time, per node)
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties

# Start broker (foreground)
bin/kafka-server-start.sh config/server.properties
```

## 3. Topic
```bash
bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092
```

## 4. Produce
```bash
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
> This is my first event
> This is my second event
# Ctrl-C to exit
```

## 5. Consume
```bash
bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```

🧪 Open consumer + producer in two terminals — events flow live.

## 6. Kafka Connect (file source/sink)
```bash
echo "plugin.path=libs/connect-file-4.2.0.jar" >> config/connect-standalone.properties
echo -e "foo\nbar" > test.txt

bin/connect-standalone.sh \
  config/connect-standalone.properties \
  config/connect-file-source.properties \
  config/connect-file-sink.properties

# Source: test.txt -> topic `connect-test`
# Sink:   topic `connect-test` -> test.sink.txt
more test.sink.txt
echo "Another line" >> test.txt   # streams through live
```

## 7. Kafka Streams (WordCount sketch)
```java
KStream<String, String> textLines = builder.stream("quickstart-events");
KTable<String, Long> wordCounts = textLines
    .flatMapValues(line -> Arrays.asList(line.toLowerCase().split(" ")))
    .groupBy((k, word) -> word)
    .count();
wordCounts.toStream().to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));
```

## 8. Teardown
```bash
rm -rf /tmp/kafka-logs /tmp/kraft-combined-logs
```

## CLI Cheatsheet
| Tool | Purpose |
|---|---|
| `kafka-storage.sh` | Generate UUID, format KRaft log dirs |
| `kafka-server-start.sh` | Boot broker (combined controller+broker in `--standalone`) |
| `kafka-topics.sh` | Create/describe/alter/delete topics |
| `kafka-console-producer.sh` | Stdin producer |
| `kafka-console-consumer.sh` | Stdout consumer |
| `connect-standalone.sh` | Single-node Connect runtime |

## Tags
[[Kafka]] [[KRaft]] [[Brokers]] [[Topics]] [[Connect]] [[Streams]]
