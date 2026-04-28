# 01 — Producer Basics

🔑 A `Producer` is a thread-safe long-lived client that batches, partitions, and sends records asynchronously; `send()/produce()` returns immediately and a delivery callback fires when the broker acks.

Source: https://kafka.apache.org/42/configuration/producer-configs/

## Lifecycle
- One `Producer` per app (thread-safe, share it).
- Build with a config dict — at minimum `bootstrap.servers` + serializers.
- `flush()` to block until queued records are sent.
- `close()` to flush + release sockets/threads. Always call on shutdown.

## Send semantics
- Java: `producer.send(record) -> Future<RecordMetadata>`.
- `confluent-kafka` (Python): `producer.produce(topic, value=..., on_delivery=cb)` is fire-and-forget; you must call `poll(0)` periodically (or `flush()`) to drive the I/O loop and trigger callbacks.
- Records buffer in an in-memory accumulator partitioned by destination, then flush to the broker as batches.

## Serializers
- `key.serializer` and `value.serializer` turn objects into `bytes`.
- Built-ins: `StringSerializer`, `ByteArraySerializer`, `IntegerSerializer`. Bring your own for JSON, Avro, Protobuf, MessagePack.
- `confluent-kafka` Python: pass `bytes` to `produce()` directly, or use `SerializingProducer` with explicit `key_serializer`/`value_serializer` callables.

## Partitioner
| Case | Behavior |
|---|---|
| Key present | Murmur2 hash of key mod num_partitions — same key always lands on the same partition (sticky-by-key). |
| Key absent | **Sticky partitioner** (default): pick one partition per batch, switch when the batch fills or `linger.ms` elapses. Improves batching vs the old round-robin. |
| Custom | `partitioner.class` for your own logic. |

⚠️ Adding partitions changes the hash mapping for keyed records — old keys may move. Plan partition counts upfront.

## Minimal example (`confluent-kafka`)
```python
from confluent_kafka import Producer

def on_delivery(err, msg) -> None:
    if err:
        logger.error(f"delivery failed: {err}")
    else:
        logger.info(f"delivered {msg.topic()}[{msg.partition()}]@{msg.offset()}")

producer = Producer({"bootstrap.servers": "localhost:9092"})
producer.produce("orders", key=b"user-42", value=b'{"amount": 99}', on_delivery=on_delivery)
producer.poll(0)        # serve callbacks
producer.flush()        # block until queue drains
```

💡 Always `flush()` before exit — buffered records are otherwise lost.
🧪 Try-it: spin up a topic with 3 partitions, send 10 keyed + 10 unkeyed records, inspect partition distribution with `kafka-console-consumer --partition`.

## Tags
[[Kafka]] [[Producers]] [[Idempotent Producer]]
