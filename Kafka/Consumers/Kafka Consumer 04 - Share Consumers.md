# 04 — Share Consumers (KIP-932)

🔑 New in 4.x: a **share group** is a queue-style consumer model where many consumers can read the **same partition** concurrently, with broker-tracked per-record acks. Decouples parallelism from partition count.

Source: https://kafka.apache.org/42/apis/

## Why
Classic consumer groups cap parallelism at `num_partitions`. Share groups remove that ceiling — useful for slow per-record work (calling external APIs, ML inference) where you want N workers per partition without splitting the topic.

## Model
- A **share group** is identified by `group.id` (separate namespace from consumer groups).
- The broker hands records to share-group members and **tracks delivery state per record** (not just per partition).
- Each fetched record is **locked** to that consumer for `share.record.lock.duration.ms` (default 30000); during the lock, no other member sees it.
- The consumer must **acknowledge** the record before the lock expires.

## Acknowledgement actions
| Action | Effect |
|---|---|
| `accept` | Record processed successfully — broker advances state, no redelivery |
| `release` | Give it back to the queue — another (or same) consumer may pick it up |
| `reject` | Permanently skip this record (poison pill) — never redelivered to the group |
| (timeout) | Lock expires ⇒ record is auto-released to the queue |

`share.acknowledgement.mode`:
- `implicit` (default) — next `poll()` accepts everything from the previous poll automatically.
- `explicit` — you must call `acknowledge()` per record; required for `release`/`reject`.

## API sketch (Java)
```java
KafkaShareConsumer<K, V> consumer = new KafkaShareConsumer<>(props);
consumer.subscribe(List.of("orders"));

while (running) {
    ConsumerRecords<K, V> records = consumer.poll(Duration.ofSeconds(1));
    for (var rec : records) {
        try {
            process(rec);
            consumer.acknowledge(rec, AcknowledgeType.ACCEPT);
        } catch (TransientError e) {
            consumer.acknowledge(rec, AcknowledgeType.RELEASE);
        } catch (PoisonError e) {
            consumer.acknowledge(rec, AcknowledgeType.REJECT);
        }
    }
    consumer.commitSync();   // flush ack state to broker
}
```

⚠️ Python wrappers (`confluent-kafka`, `aiokafka`) are catching up — check release notes; expect parity in the next minor releases. As of 4.2, the Java client is the reference.

## Share group vs consumer group
| | Consumer Group | Share Group |
|---|---|---|
| Parallelism cap | `num_partitions` | unbounded |
| Order guarantee | per partition | none |
| Offset tracking | one offset per partition | per-record delivery state on broker |
| Failure mode | rebalance reassigns the partition | lock expires, record redelivered |
| Use case | event sourcing, stream processing, replay | task queue, work distribution |
| Mental model | log reader | message queue (RabbitMQ-like) |

## Configs
| Config | Default | Notes |
|---|---|---|
| `share.record.lock.duration.ms` | `30000` | Per-record lock window. Up to ~60s broker-enforced max. |
| `share.acknowledgement.mode` | `implicit` | `explicit` to use release/reject. |
| `share.acquire.mode` | `batch_optimized` | `record_limit` to honor `max.poll.records` strictly. |

💡 Share groups are not a drop-in for stream processing — order is gone. Use them for "process this independent task" workloads.

## Tags
[[Kafka]] [[Consumers]] [[Share Groups]] [[Consumer Groups]]
