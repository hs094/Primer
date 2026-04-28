# 01 — Client Languages and Libraries

🔑 The Java `kafka-clients` jar is the reference implementation; everything else is a wrapper or reimplementation. For production non-JVM use: pick the `librdkafka`-backed binding when available.

Source: https://cwiki.apache.org/confluence/display/KAFKA/Clients (canonical client list)

## Tier map
| Tier | What it means |
|---|---|
| **Reference** | Java `kafka-clients` — every protocol feature lands here first |
| **librdkafka-based** | C library + thin language bindings — full feature parity, fastest non-JVM option |
| **Native reimpl** | Pure language reimplementations of the wire protocol — feature lag, but no native deps |

## Java
- **`org.apache.kafka:kafka-clients`** (Apache, official). All features (transactions, idempotence, share consumer, admin API). Only client guaranteed at-feature-parity with the broker.

## Python
| Library | Backend | Async? | Transactions | Notes |
|---|---|---|---|---|
| **`confluent-kafka`** | librdkafka (C) | sync + thread-callback | yes | Confluent's official; fastest; full feature parity. **Default choice.** |
| **`aiokafka`** | pure Python | native `asyncio` | partial (older versions) | Best DX for async apps; lags broker features by 1-2 releases |
| **`kafka-python`** | pure Python | sync only | no | Legacy, mostly unmaintained relative to the others; avoid for new code |

```python
# confluent-kafka — production default
from confluent_kafka import Producer, Consumer

# aiokafka — async-native
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
```

## Go
| Library | Backend | Notes |
|---|---|---|
| **`confluent-kafka-go`** | librdkafka (cgo) | Confluent official; full features; cgo dependency |
| **`twmb/franz-go`** | pure Go | Modern, fast, idiomatic; widely adopted; no cgo. Strong choice if you want a static binary. |
| **`segmentio/kafka-go`** | pure Go | Older pure-Go client; simpler API but less complete |

## Node.js / JavaScript
- **`kafkajs`** — pure JS, the de facto choice. Good DX, supports transactions and idempotence; no librdkafka binding maintained for Node.
- `node-rdkafka` exists but is less popular and lags.

## Other languages (highlights)
| Language | Library |
|---|---|
| Rust | `rdkafka` (librdkafka), `rskafka` (pure Rust) |
| C# / .NET | `Confluent.Kafka` (librdkafka) |
| C / C++ | `librdkafka` directly |
| Ruby | `rdkafka-ruby`, `ruby-kafka` |
| PHP | `php-rdkafka` |
| Erlang/Elixir | `brod`, `kafka_ex` |

## How to choose
- **JVM app** → `kafka-clients`. Done.
- **Python, performance-sensitive or needs transactions** → `confluent-kafka`.
- **Python, async-first FastAPI app, no transactions** → `aiokafka`.
- **Go, want static binary** → `franz-go`.
- **Need brand-new broker feature (e.g. share consumer)** → check if the binding has it; otherwise wait or use Java.

⚠️ "Pure-X" clients reimplement the wire protocol — every new KIP is a port. Lag is normal. For bleeding-edge features, librdkafka or the Java client land first.

💡 The cwiki Clients page (https://cwiki.apache.org/confluence/display/KAFKA/Clients) is the canonical, community-maintained list — broader than this note.

## Tags
[[Kafka]] [[Producers]] [[Consumers]]
