# 04 — Connect Administration

🔑 Rolling restart with cooperative rebalancing, secrets via config providers, opt-in exactly-once for sources.

Source: https://kafka.apache.org/42/kafka-connect/administration/

## Rolling restart
- Distributed Connect uses **incremental cooperative rebalancing** (default since 2.3).
- Restart workers one at a time; the cluster only reassigns *that* worker's tasks, leaving others untouched.
- Knob: `scheduled.rebalance.max.delay.ms` (default `300000` = 5 min) — when a worker disappears, the cluster waits this long before redistributing its tasks. If the worker rejoins within the delay, it picks up its old assignments.
- Procedure:
  1. `PUT /connectors/{name}/pause` (optional, for very latency-sensitive sinks).
  2. Stop worker → upgrade → start worker → wait for `RUNNING`.
  3. Move to next worker.
  4. Resume any paused connectors at the end.

## Secret externalization — config providers
- Avoid plaintext secrets in connector config / REST payloads.
- Worker-level providers replace `${provider:path:key}` placeholders at runtime.

```properties
# worker config
config.providers=file,env
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider
config.providers.env.class=org.apache.kafka.common.config.provider.EnvVarConfigProvider
```

```properties
# connector config — secrets stay on disk / in env
database.user=${file:/etc/connect/secrets.properties:db.user}
database.password=${file:/etc/connect/secrets.properties:db.password}
api.token=${env:DEBEZIUM_API_TOKEN}
```

| Provider class | Source | Placeholder |
| --- | --- | --- |
| `FileConfigProvider` | Properties file on disk | `${file:<path>:<key>}` |
| `EnvVarConfigProvider` | Process environment | `${env:<VAR>}` |
| `DirectoryConfigProvider` | One file per key in a directory | `${directory:<dir>:<key>}` |
| Custom | Vault, AWS SM, GCP SM, etc. (community) | `${myvault:secret/foo:password}` |

⚠️ `GET /connectors/{name}/config` returns the *raw* placeholders, not resolved values — fine for inspection, no leakage. Validate this in your fork/connector if you log configs.

## Exactly-once source connectors
- Off by default. Enable cluster-wide:

```properties
# worker config (distributed)
exactly.once.source.support=enabled       # was 'preparing' in upgrade phase
```

- Requires:
  - Brokers ≥ 2.6, ideally newer (Kafka 4.x is fine).
  - The connector class must override `exactlyOnceSupport()` to declare `SUPPORTED` (and optionally `canDefineTransactionBoundaries()`).
  - Each connector gets its own producer transactional id; Connect coordinates commits with the offset topic.
- Per-record cost: an extra commit round-trip. Like Streams EOS, it covers Kafka writes only — external side effects must still be idempotent.
- Sinks: there is no symmetric flag. "Exactly-once into a sink" is a property of the *sink system* (idempotent upserts, deduplication keys, etc.).

## Graceful task migration on rebalance
- Cooperative rebalancing means a task in flight on worker A is **not** revoked instantly when worker B joins — only newly-needed reassignments move.
- A failed worker's tasks are held back until `scheduled.rebalance.max.delay.ms` to absorb transient outages.
- Pause vs Stop:
  - `PUT /pause` — keep tasks alive but idle (resume is fast).
  - `PUT /stop` — release resources entirely (resume rebuilds tasks from scratch).
- Connector / task states cycle through `UNASSIGNED → RUNNING ↔ PAUSED ↔ STOPPED`, with `FAILED` and `RESTARTING` as exceptional states. Status updates flow through `status.storage.topic` and may lag a few hundred ms.

💡 In CI/CD, target `POST /connectors/{name}/restart?includeTasks=true&onlyFailed=true` after deploys — it heals failed tasks without disturbing healthy ones.

## Tags
[[Kafka]] [[Kafka Connect]] [[Connector]] [[Exactly-Once]]
