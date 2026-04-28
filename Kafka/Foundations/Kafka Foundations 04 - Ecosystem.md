# 04 — Ecosystem

🔑 The official ecosystem page is a stub — the canonical registry of Kafka-adjacent tools is the cwiki Ecosystem page.

Source: https://kafka.apache.org/42/getting-started/ecosystem/

⚠️ The 4.2 doc page only states there are "a plethora of tools that integrate with Kafka outside the main distribution" and points at the wiki. Canonical list: https://cwiki.apache.org/confluence/display/KAFKA/Ecosystem

## Categories Called Out
- Stream processing systems
- Hadoop integration
- Monitoring
- Deployment tools

## Practical Ecosystem Map (synthesized)
| Category | Notable Projects |
|---|---|
| **Clients (non-JVM)** | `librdkafka` (C/C++), `confluent-kafka-python`, `aiokafka`, `kafka-go`, `node-rdkafka`, `sarama` (Go) |
| **Schema / serialization** | Confluent Schema Registry, Apicurio Registry, Avro, Protobuf, JSON Schema |
| **Stream processing** | Kafka Streams (in-tree), ksqlDB, Apache Flink, Apache Spark Structured Streaming, Apache Samza, Apache Beam |
| **Connectors** | Confluent Hub, Debezium (CDC), Kafka Connect file/JDBC/S3/Elastic/Mongo connectors |
| **Management / UI** | AKHQ, Kafka UI (Provectus), Confluent Control Center, Redpanda Console, Conduktor |
| **Kubernetes operators** | Strimzi, Confluent for Kubernetes, Koperator |
| **Replication / mirroring** | MirrorMaker 2 (in-tree), Confluent Replicator, Cluster Linking |
| **Monitoring** | JMX Exporter + Prometheus, Cruise Control, Kafka Lag Exporter, Burrow |
| **Distributions / managed** | Confluent Platform, Confluent Cloud, Amazon MSK, Aiven, Redpanda (compatible re-impl) |
| **Security** | Kafka native SASL/SSL/ACLs, Open Policy Agent integrations |

💡 For a Python-first stack: `confluent-kafka` (librdkafka binding) for clients, Schema Registry for contracts, Debezium for CDC, Strimzi if you're on Kubernetes.

## Tags
[[Kafka]] [[Connect]] [[Streams]] [[KRaft]] [[Brokers]]
