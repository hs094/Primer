# 03 — Transactions

🔑 Transactions atomically commit a set of writes (across partitions and topics) plus consumer offsets, surviving producer restarts via a fixed `transactional.id`. Pair with `isolation.level=read_committed` consumers for end-to-end EOS.

Source: https://kafka.apache.org/42/configuration/producer-configs/

## Config
- `transactional.id` — stable string identifying this logical producer (e.g. `"orders-processor-1"`). Recovers in-flight transactions on restart by fencing the old PID epoch.
- `enable.idempotence=true` (auto-enabled).
- `transaction.timeout.ms` (default 60000) — coordinator aborts if no progress.

## API (Java names; Python wrappers mirror these)
| Method | Purpose |
|---|---|
| `init_transactions()` | one-time per producer; registers `transactional.id` with coordinator, fences zombies |
| `begin_transaction()` | starts a new txn |
| `produce(...)` | normal sends — buffered in the open txn |
| `send_offsets_to_transaction(offsets, group_metadata)` | atomically commit consumer offsets *as part of* this txn |
| `commit_transaction()` | flush + write commit marker; readers (`read_committed`) now see the records |
| `abort_transaction()` | discard buffered records |

## Read-process-write pattern
```python
from confluent_kafka import Producer, Consumer, TopicPartition

producer = Producer({
    "bootstrap.servers": "localhost:9092",
    "transactional.id": "orders-enricher-1",
})
consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "orders-enricher",
    "enable.auto.commit": False,
    "isolation.level": "read_committed",
})
consumer.subscribe(["orders.raw"])
producer.init_transactions()

while True:
    msgs = consumer.consume(num_messages=500, timeout=1.0)
    if not msgs:
        continue
    producer.begin_transaction()
    try:
        for m in msgs:
            producer.produce("orders.enriched", value=enrich(m.value()))
        # commit offsets *inside* the txn
        offsets = [TopicPartition(m.topic(), m.partition(), m.offset() + 1) for m in msgs]
        producer.send_offsets_to_transaction(offsets, consumer.consumer_group_metadata())
        producer.commit_transaction()
    except Exception:
        producer.abort_transaction()
        raise
```

## Consumer side
- Set `isolation.level=read_committed` — consumer skips records belonging to aborted txns and waits past the **Last Stable Offset (LSO)** for in-flight ones (small added latency).
- `read_uncommitted` (default) sees everything, including aborted records.

⚠️ Gotchas
- Two producers with the same `transactional.id` → newer one fences older (older's writes start failing with `ProducerFenced`). Useful for HA failover but lethal for accidental dupes.
- Long-running txns block consumers reading committed data → keep them short.
- `transactional.id` should be stable per logical producer instance (e.g. partition-bound), not random per process.

💡 If you're not doing read-process-write, you usually don't need transactions — idempotence covers single-producer-write EOS.

## Tags
[[Kafka]] [[Producers]] [[Transactions]] [[Idempotent Producer]]
