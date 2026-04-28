# 02 — Connect User Guide

🔑 Drive everything via REST; in distributed mode, three Kafka topics hold all cluster state.

Source: https://kafka.apache.org/42/kafka-connect/user-guide/

## Worker config — distributed mode

```properties
bootstrap.servers=broker1:9092,broker2:9092
group.id=connect-prod                       # cluster identity; do not share across clusters
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter

config.storage.topic=connect-configs        # 1 partition, compacted
offset.storage.topic=connect-offsets        # 25+ partitions, compacted
status.storage.topic=connect-status         # ~5 partitions, compacted

config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

plugin.path=/opt/connectors                 # colon-separated dirs of connector JARs
listeners=http://0.0.0.0:8083
```

- **`group.id` collision** between two Connect clusters on the same brokers will silently merge them — pick unique values.
- Three storage topics must exist (auto-created if you have permissions). They are **compacted** so only the latest state per key survives.

## Standalone mode
- Single process: `connect-standalone.sh worker.properties connector1.properties [connector2.properties …]`
- Offsets stored in a local file (`offset.storage.file.filename`).
- No REST cluster — single-node only.

## Connectors vs tasks
- A **connector** is a logical pipe (one config: source/sink class + connection params + topic).
- The connector class splits work into **tasks** at runtime (`tasks.max` = upper bound).
  - JDBC source: 1 task per table.
  - Sink: 1 task per N input partitions, capped at `tasks.max`.
- Tasks are the unit of parallelism and what gets balanced across workers.

```
connector "orders-jdbc"  →  tasks [t0, t1, t2]  →  spread across worker[A, B]
```

## REST API — common endpoints

| Method & path | Purpose |
| --- | --- |
| `GET /connectors` | List connector names |
| `POST /connectors` | Create (body: `{"name": ..., "config": {...}}`) |
| `GET /connectors/{name}` | Show config + tasks |
| `GET /connectors/{name}/status` | Connector + per-task state (`RUNNING` / `FAILED` / `PAUSED` / `STOPPED`) |
| `GET /connectors/{name}/config` | Current config |
| `PUT /connectors/{name}/config` | Create-or-update config (idempotent) |
| `POST /connectors/{name}/restart` | Restart the connector (and optionally tasks via `?includeTasks=true&onlyFailed=true`) |
| `POST /connectors/{name}/tasks/{id}/restart` | Restart a single task |
| `PUT /connectors/{name}/pause` | Pause (resources kept) |
| `PUT /connectors/{name}/stop` | Stop (resources released) |
| `PUT /connectors/{name}/resume` | Resume |
| `DELETE /connectors/{name}` | Delete (does not wipe offsets) |
| `GET /connector-plugins` | List installed plugin classes |
| `PUT /connector-plugins/{class}/config/validate` | Validate a candidate config |

```python
# Example with confluent-kafka isn't relevant here — Connect is REST.
import httpx

cfg = {
    "name": "pg-source",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "tasks.max": "1",
        "database.hostname": "pg",
        "database.user": "${file:/etc/connect/secrets.properties:pg.user}",
        "database.password": "${file:/etc/connect/secrets.properties:pg.password}",
        "topic.prefix": "pg",
    },
}
r = httpx.put("http://connect:8083/connectors/pg-source/config", json=cfg["config"])
r.raise_for_status()
```

💡 Prefer `PUT …/config` over `POST /connectors` in CI — `PUT` is idempotent and works for both create and update.

## Tags
[[Kafka]] [[Kafka Connect]] [[Connector]]
