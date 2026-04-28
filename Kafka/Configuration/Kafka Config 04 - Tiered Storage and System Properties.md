# 04 — Tiered Storage and System Properties

🔑 Tiered Storage offloads cold log segments to object storage (S3, GCS, Azure Blob) so brokers keep only a hot working set on disk. System properties are JVM-level `-D` flags for security and runtime knobs.

Sources:
- https://kafka.apache.org/42/configuration/tiered-storage-configs/
- https://kafka.apache.org/42/configuration/system-properties/

## Tiered Storage — Cluster-Level

| key | type | default | meaning |
| --- | --- | --- | --- |
| `remote.log.storage.system.enable` | boolean | false | Master switch. Must be true on every broker before any topic can tier. ⚠️ Static — restart required. |
| `remote.log.storage.manager.class.name` | string | null | Plugin implementing `RemoteStorageManager` (S3, GCS, etc.). Vendor-specific JAR on classpath. |
| `remote.log.metadata.manager.class.name` | string | `TopicBasedRemoteLogMetadataManager` | Where remote-segment metadata lives. Default uses an internal Kafka topic (`__remote_log_metadata`). |
| `remote.log.manager.task.interval.ms` | long | 30000 (30s) | How often the RLM thread runs copy/expire tasks. |
| `remote.log.manager.thread.pool.size` | int | 2 | RLM scheduler thread pool. |
| `remote.log.reader.threads` | int | 10 | Threads serving reads from remote storage. Bump if many consumers fetch cold data. |
| `remote.fetch.max.wait.ms` | int | 500 | Max time the broker waits before responding to a fetch that hits remote storage. |
| `remote.log.index.file.cache.total.size.bytes` | long | 1 GiB | Local cache for offset/time indexes downloaded from remote. |
| `remote.log.manager.copy.max.bytes.per.second` | long | Long.MAX | Throttle for local→remote upload. Use to cap network impact. |
| `remote.log.manager.fetch.max.bytes.per.second` | long | Long.MAX | Throttle for remote→local fetch. |
| `remote.log.metadata.topic.replication.factor` | short | 3 | RF of `__remote_log_metadata`. Cluster needs ≥3 brokers. |
| `remote.log.metadata.topic.num.partitions` | int | 50 | Partition count of metadata topic. Set once — cannot grow without re-init. |

## Tiered Storage — Per-Topic

| key | type | default | meaning |
| --- | --- | --- | --- |
| `remote.storage.enable` | boolean | false | Turn on tiering for this topic. |
| `local.retention.ms` | long | -2 | How long segments stay on local disk after being uploaded. `-2` = inherit `retention.ms` (no eager eviction). |
| `local.retention.bytes` | long | -2 | Local-disk size cap per partition. `-2` = inherit `retention.bytes`. |

💡 Common pattern: `retention.ms=infinite` (or 30d), `local.retention.ms=86400000` (1d hot on local NVMe), and let object storage carry the historical tail.

⚠️ Once a topic has `remote.storage.enable=true` you cannot toggle it off without recreating the topic. Plan accordingly.

## System Properties (`-D` JVM flags)

Set these on broker/client JVM startup via `KAFKA_OPTS` or directly in the launch script.

### Security

| property | meaning |
| --- | --- |
| `-Djava.security.auth.login.config=/path/jaas.conf` | JAAS config for SASL (Kerberos, PLAIN, OAUTHBEARER). Required when `security.protocol=SASL_*`. |
| `-Dorg.apache.kafka.allowed.login.modules=...` | Allowlist of SASL JAAS login modules (since 4.2). Preferred over the deprecated disallowed-list below. |
| `-Dorg.apache.kafka.disallowed.login.modules=com.sun.security.auth.module.JndiLoginModule,...` | Legacy denylist (CVE-2023-25194 mitigation). ⚠️ Deprecated in 4.2. |
| `-Dorg.apache.kafka.automatic.config.providers=none` | Disable auto-loading of built-in `ConfigProvider` plugins (CVE-2024-31141). Set to `none` in untrusted-config environments. |
| `-Dorg.apache.kafka.sasl.oauthbearer.allowed.urls=https://idp.example.com/...` | Allowlist of token/JWKS endpoints for SASL OAUTHBEARER. Default empty — must set explicitly. |
| `-Dorg.apache.kafka.sasl.oauthbearer.allowed.files=/tmp/token,/tmp/key.pem` | Allowlist of files OAUTHBEARER plugin can read. Default empty. |

### Networking

| property | meaning |
| --- | --- |
| `-Djava.net.preferIPv4Stack=true` | Force IPv4. Common on networks where dual-stack DNS misbehaves. |

### GC / JVM (worth knowing)

| property | meaning |
| --- | --- |
| `-Xms` / `-Xmx` | Heap size. Brokers commonly run 6–16 GiB heap; rest of RAM = page cache. |
| `-XX:+UseG1GC` | Default since JDK 9. Suitable for Kafka's allocation profile. |
| `-XX:MaxGCPauseMillis=20` | G1 pause-time goal. Kafka recommends ~20ms. |
| `-XX:InitiatingHeapOccupancyPercent=35` | Trigger concurrent GC earlier — avoids long mixed pauses. |
| `-XX:+ExplicitGCInvokesConcurrent` | Make `System.gc()` calls (e.g. from MMap cleaner) concurrent. |

💡 Don't oversize the heap — Kafka's hot path is page cache. 6 GiB heap on a 64 GiB node is normal.

## Tags
[[Kafka]] [[Tiered Storage]] [[Brokers]]
