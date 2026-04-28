# Kafka Knowledge Pack

Crisp, scannable revision notes covering Apache Kafka 4.2 documentation.
Source: https://kafka.apache.org/42/

## How to use
- Each note = one concept, optimized for recall.
- 🔑 = core idea on every note's first line.
- ⚠️ = gotcha, 💡 = tip, 🧪 = try-it.
- Code blocks are minimum that exercises the feature.
- Python clients via `confluent-kafka` (sync, full feature parity) or `aiokafka` (async). Streams DSL examples are Java (Streams is JVM-only).

## Foundations (`/getting-started/`)

| # | Topic | Note |
|---|---|---|
| 01 | What Kafka is — event streaming platform | [[Foundations/Kafka Foundations 01 - Overview]] |
| 02 | Use cases — messaging, metrics, logs, CDC, ES | [[Foundations/Kafka Foundations 02 - Use Cases]] |
| 03 | Quickstart — KRaft bootstrap, CLI tour | [[Foundations/Kafka Foundations 03 - Quickstart]] |
| 04 | Ecosystem — clients, schema, processors, UIs | [[Foundations/Kafka Foundations 04 - Ecosystem]] |
| 05 | The 6 APIs (Producer/Consumer/Share/Streams/Connect/Admin) | [[Foundations/Kafka Foundations 05 - APIs]] |

## Brokers — Design + Implementation (`/design/`, `/implementation/`)

| # | Topic | Note |
|---|---|---|
| 01 | Persistence — page cache, sequential I/O | [[Brokers/Kafka Brokers 01 - Persistence and Storage]] |
| 02 | Efficiency — `sendfile` zero-copy, batching, compression | [[Brokers/Kafka Brokers 02 - Efficiency and Zero-Copy]] |
| 03 | Replication & ISR — leader election, acks, min.insync | [[Brokers/Kafka Brokers 03 - Replication and ISR]] |
| 04 | Log compaction — per-key retention, tombstones | [[Brokers/Kafka Brokers 04 - Log Compaction]] |
| 05 | Quotas — bandwidth + request-rate, throttling | [[Brokers/Kafka Brokers 05 - Quotas]] |
| 06 | Network layer — Reactor, NIO, request handling | [[Brokers/Kafka Brokers 06 - Network Layer]] |
| 07 | Messages & RecordBatch — on-disk format | [[Brokers/Kafka Brokers 07 - Messages and Record Batch]] |
| 08 | Log format — segments, .log/.index/.timeindex | [[Brokers/Kafka Brokers 08 - Log Format]] |
| 09 | Distribution — controller, KRaft, partition assignment | [[Brokers/Kafka Brokers 09 - Distribution]] |
| 10 | Delivery semantics & transactions — EOS, share groups | [[Brokers/Kafka Brokers 10 - Delivery Semantics and Transactions]] |

## Producers (`/configuration/producer-configs/`)

| # | Topic | Note |
|---|---|---|
| 01 | Basics — `Producer.produce()`, serializers, partitioner | [[Producers/Kafka Producer 01 - Basics]] |
| 02 | Acks & idempotence — durability tradeoffs | [[Producers/Kafka Producer 02 - Acks and Idempotence]] |
| 03 | Transactions — `transactional.id`, read-process-write | [[Producers/Kafka Producer 03 - Transactions]] |
| 04 | Key configs — the configs you actually tune | [[Producers/Kafka Producer 04 - Key Configs]] |

## Consumers (`/configuration/consumer-configs/`)

| # | Topic | Note |
|---|---|---|
| 01 | Basics & groups — subscribe/assign, poll loop | [[Consumers/Kafka Consumer 01 - Basics and Groups]] |
| 02 | Offsets & commits — auto vs manual, reset policy | [[Consumers/Kafka Consumer 02 - Offsets and Commits]] |
| 03 | Rebalancing — cooperative sticky, static membership | [[Consumers/Kafka Consumer 03 - Rebalancing]] |
| 04 | Share consumers (NEW in 4.x) — queue semantics | [[Consumers/Kafka Consumer 04 - Share Consumers]] |

## Clients

| # | Topic | Note |
|---|---|---|
| 01 | Languages & libraries — Java, Python, Go, Node | [[Clients/Kafka Clients 01 - Languages and Libraries]] |

## Configuration (`/configuration/`)

| # | Topic | Note |
|---|---|---|
| 01 | Broker configs — listeners, log.dirs, KRaft | [[Configuration/Kafka Config 01 - Broker Configs]] |
| 02 | Topic configs — retention, cleanup.policy | [[Configuration/Kafka Config 02 - Topic Configs]] |
| 03 | Group + Streams configs — protocols, EOS | [[Configuration/Kafka Config 03 - Group and Streams Configs]] |
| 04 | Tiered storage + system properties | [[Configuration/Kafka Config 04 - Tiered Storage and System Properties]] |
| 05 | Configuration providers — `${file:...}`, `${env:...}` | [[Configuration/Kafka Config 05 - Configuration Providers]] |

## Operations (`/operations/`)

| # | Topic | Note |
|---|---|---|
| 01 | Basic ops — kafka-topics/configs/reassign, graceful shutdown | [[Operations/Kafka Operations 01 - Basic Operations]] |
| 02 | KRaft — controller quorum, `kafka-storage.sh format` | [[Operations/Kafka Operations 02 - KRaft]] |
| 03 | Hardware & OS — disks, RAM, JVM, kernel tunables | [[Operations/Kafka Operations 03 - Hardware and OS]] |
| 04 | Monitoring — JMX metrics, Prometheus, consumer lag | [[Operations/Kafka Operations 04 - Monitoring]] |
| 05 | Geo replication — MirrorMaker 2, active/active | [[Operations/Kafka Operations 05 - Geo Replication]] |
| 06 | Multi-tenancy — namespacing, ACLs, quotas | [[Operations/Kafka Operations 06 - Multi-Tenancy]] |
| 07 | Tiered storage — hot/cold split, S3 plugin | [[Operations/Kafka Operations 07 - Tiered Storage]] |
| 08 | Consumer rebalance protocol — KIP-848 server-side | [[Operations/Kafka Operations 08 - Consumer Rebalance Protocol]] |

## Security (`/security/`)

| # | Topic | Note |
|---|---|---|
| 01 | Overview & listeners — protocol map, advertised | [[Security/Kafka Security 01 - Overview and Listeners]] |
| 02 | SSL & SASL — keystore, mTLS, SCRAM, OAUTHBEARER | [[Security/Kafka Security 02 - SSL and SASL]] |
| 03 | Authorization & ACLs — StandardAuthorizer, kafka-acls | [[Security/Kafka Security 03 - Authorization and ACLs]] |

## Kafka Streams (`/streams/`)

| # | Topic | Note |
|---|---|---|
| 01 | Overview — client library, no separate cluster | [[Streams/Kafka Streams 01 - Overview]] |
| 02 | Core concepts — KStream/KTable/duality, topology | [[Streams/Kafka Streams 02 - Core Concepts]] |
| 03 | DSL — StreamsBuilder, stateless + stateful ops | [[Streams/Kafka Streams 03 - DSL]] |
| 04 | State stores — RocksDB, changelog, standbys | [[Streams/Kafka Streams 04 - State Stores]] |
| 05 | Windowing — tumbling, hopping, sliding, session | [[Streams/Kafka Streams 05 - Windowing]] |
| 06 | Joins — co-partitioning, GlobalKTable, FK joins | [[Streams/Kafka Streams 06 - Joins]] |
| 07 | Exactly-once — `exactly_once_v2`, EOS internals | [[Streams/Kafka Streams 07 - Exactly-Once]] |

## Kafka Connect (`/kafka-connect/`)

| # | Topic | Note |
|---|---|---|
| 01 | Overview — framework, source/sink, converters | [[Connect/Kafka Connect 01 - Overview]] |
| 02 | User guide — REST API, distributed worker, tasks | [[Connect/Kafka Connect 02 - User Guide]] |
| 03 | Connector development — SourceTask/SinkTask, SMTs | [[Connect/Kafka Connect 03 - Connector Development]] |
| 04 | Administration — rolling restart, EOS sources | [[Connect/Kafka Connect 04 - Administration]] |

## Tags
[[Kafka]] [[Event Streaming]] [[KRaft]] [[Brokers]] [[Producers]] [[Consumers]] [[Kafka Streams]] [[Kafka Connect]] [[Replication]] [[ISR]] [[Idempotent Producer]] [[Transactions]] [[Share Groups]] [[Log Compaction]]
