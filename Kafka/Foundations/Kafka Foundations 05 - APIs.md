# 05 — APIs

🔑 Six APIs in 4.2: Producer, Consumer, Share Consumer (new), Streams, Connect, Admin — all on `kafka-clients`/`kafka-streams` artifacts.

Source: https://kafka.apache.org/42/apis/

## API Matrix
| API | Purpose | Artifact | Entrypoint |
|---|---|---|---|
| **Producer** | Send streams to topics | `org.apache.kafka:kafka-clients:4.2.0` | `org.apache.kafka.clients.producer.KafkaProducer` |
| **Consumer** | Read streams from topics (partition assignment) | `org.apache.kafka:kafka-clients:4.2.0` | `org.apache.kafka.clients.consumer.KafkaConsumer` |
| **Share Consumer** ✨ | Cooperative queue-like consumption with per-record ack | `org.apache.kafka:kafka-clients:4.2.0` | `org.apache.kafka.clients.consumer.KafkaShareConsumer` |
| **Streams** | Topic-to-topic transforms, joins, windowing, state | `org.apache.kafka:kafka-streams:4.2.0` | `org.apache.kafka.streams.KafkaStreams` |
| **Connect** | Build source/sink connectors | (in distribution) | `org.apache.kafka.connect.*` interfaces |
| **Admin** | Manage topics, brokers, ACLs, configs | `org.apache.kafka:kafka-clients:4.2.0` | `org.apache.kafka.clients.admin.Admin` |

## Share Consumer: What's Different
⚠️ Share Consumer is a new abstraction in 4.x — fundamentally different model from the classic Consumer.

| Aspect | Consumer (classic) | Share Consumer |
|---|---|---|
| Assignment | Partition pinned to one consumer in group | Records distributed across share group, no partition pinning |
| Acknowledgement | Offset commits (per partition) | Per-record ack/release/reject |
| Semantics | Stream/log replay | Queue-like, work-queue dispatch |
| Re-delivery | Re-consume from offset | Per-record redelivery on failure |

💡 Use share consumers when you want classic queue semantics (multiple workers, any worker takes any record) without forcing 1 partition = 1 worker.

## Python Skeletons (preferred client lib)

```python
# Producer — confluent-kafka
from confluent_kafka import Producer

producer = Producer({"bootstrap.servers": "localhost:9092"})
producer.produce("quickstart-events", key="alice", value=f"hello {n}")
producer.flush()
```

```python
# Consumer — confluent-kafka
from confluent_kafka import Consumer

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "demo",
    "auto.offset.reset": "earliest",
})
consumer.subscribe(["quickstart-events"])
while True:
    msg = consumer.poll(1.0)
    if msg and not msg.error():
        print(f"{msg.key()}: {msg.value()}")
```

```python
# Async producer/consumer — aiokafka
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer

async def main():
    producer = AIOKafkaProducer(bootstrap_servers="localhost:9092")
    await producer.start()
    try:
        await producer.send_and_wait("quickstart-events", b"hello")
    finally:
        await producer.stop()
```

```python
# Admin — confluent-kafka
from confluent_kafka.admin import AdminClient, NewTopic

admin = AdminClient({"bootstrap.servers": "localhost:9092"})
admin.create_topics([NewTopic("orders", num_partitions=6, replication_factor=3)])
```

⚠️ Streams + Share Consumer + Connect have no first-class Python equivalent — drop to JVM (Kotlin/Scala/Java) for those.

## Tags
[[Kafka]] [[Producers]] [[Consumers]] [[Streams]] [[Connect]] [[Brokers]]
