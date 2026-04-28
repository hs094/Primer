# 02 — Offsets and Commits

🔑 Each consumer group tracks the next offset to read per partition in the internal `__consumer_offsets` topic. **When** you commit (before vs after processing) is the entire knob between at-most-once and at-least-once delivery.

Source: https://kafka.apache.org/42/configuration/consumer-configs/

## Where offsets live
- `__consumer_offsets` — compacted internal topic, 50 partitions by default.
- Keyed by `(group.id, topic, partition)` → latest committed offset wins.
- The **group coordinator** owns reads/writes for a given group.

## Commit modes
| Mode | Config | When | Trade-off |
|---|---|---|---|
| Auto-commit | `enable.auto.commit=true` (default), `auto.commit.interval.ms=5000` | Background, on `poll()` | Easy, but you can lose work or replay on crash — commit may land before processing finishes |
| Manual sync | `enable.auto.commit=false`, `consumer.commit()` (sync) | After processing | Blocks until ack; safest for at-least-once |
| Manual async | `consumer.commit(asynchronous=True)` | After processing | Faster, but errors are silent unless you handle the callback |

## Delivery semantics
| Pattern | Order | Result |
|---|---|---|
| Process → commit | crash before commit ⇒ replay on restart | **at-least-once** (duplicates possible) |
| Commit → process | crash after commit, before write ⇒ record lost | **at-most-once** (gaps possible) |
| Transactional read-process-write | offsets + writes commit atomically | **exactly-once** (see [[Transactions]]) |

## auto.offset.reset
Applies only when no committed offset exists for the group:
| Value | Behavior |
|---|---|
| `latest` (default) | Start from end — only see new records |
| `earliest` | Read from beginning of retained log |
| `none` | Throw `NoOffsetForPartitionException` — fail loudly |
| `by_duration:PT1H` (4.x) | Start from records produced in the last hour |

⚠️ `auto.offset.reset` does **not** trigger if you have committed offsets. Reset by deleting offsets (`kafka-consumer-groups --reset-offsets`) or using a fresh `group.id`.

## Manual commit example
```python
consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "billing",
    "enable.auto.commit": False,
})
consumer.subscribe(["invoices"])

while True:
    msg = consumer.poll(1.0)
    if msg is None or msg.error():
        continue
    process(msg.value())          # do work first
    consumer.commit(msg, asynchronous=False)   # then commit this offset+1
```

💡 Commit in batches (every N records or every T seconds), not per-record — `__consumer_offsets` writes aren't free.

⚠️ Auto-commit commits the last `poll()`-returned offset on the *next* `poll()` — so a crash mid-processing replays from the last commit, not from where you stopped. If you can't tolerate dupes, you need transactions or an idempotent sink.

## Tags
[[Kafka]] [[Consumers]] [[Offsets]] [[Consumer Groups]] [[Transactions]]
