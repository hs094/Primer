# 01 — Consumer Basics and Groups

🔑 A `Consumer` joins a **consumer group** identified by `group.id`; the broker assigns partitions across group members so each partition is consumed by exactly one member at a time. `poll()` drives fetches, heartbeats, and rebalances.

Source: https://kafka.apache.org/42/configuration/consumer-configs/

## Group model
- All consumers sharing a `group.id` form one logical reader; the broker splits partitions among them.
- **One partition → one consumer-in-group max.** Spare consumers (more consumers than partitions) sit idle.
- Different `group.id`s read independent copies of the topic — pub/sub fan-out.
- Group state (membership, offsets) lives on a **group coordinator** broker.

## subscribe vs assign
| API | Behavior | Use when |
|---|---|---|
| `subscribe([topics])` | Joins group, broker assigns partitions, rebalances on membership change | normal apps; you want HA + scaling |
| `assign([TopicPartition,...])` | Manual, no group, no rebalance, no auto offset commits | precise control, replay tools, single-process pipelines |

⚠️ Don't mix: a consumer either subscribes *or* assigns, not both.

## The poll loop
- `poll(timeout)` returns a batch of records (up to `max.poll.records`, default 500).
- It also: sends heartbeats, runs rebalance callbacks, fetches metadata. **You must call it regularly** — silence longer than `max.poll.interval.ms` (default 5 min) gets you kicked from the group.
- Single-threaded per consumer instance.

## Minimal example (`confluent-kafka`)
```python
from confluent_kafka import Consumer, KafkaError

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "order-processor",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": True,   # default
})
consumer.subscribe(["orders"])

try:
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            raise KafkaException(msg.error())
        process(msg.value())
finally:
    consumer.close()  # leaves group cleanly, triggers fast rebalance
```

## Async equivalent (`aiokafka`)
```python
from aiokafka import AIOKafkaConsumer

consumer = AIOKafkaConsumer(
    "orders",
    bootstrap_servers="localhost:9092",
    group_id="order-processor",
    auto_offset_reset="earliest",
)
await consumer.start()
try:
    async for msg in consumer:
        await process(msg.value)
finally:
    await consumer.stop()
```

💡 `consumer.close()` is critical — without it you wait `session.timeout.ms` (45s) before the group rebalances.

🧪 Try-it: start two consumers in the same group on a 4-partition topic. Kill one. Watch partitions migrate via broker logs.

## Tags
[[Kafka]] [[Consumers]] [[Consumer Groups]] [[Offsets]] [[Rebalance]]
