# SH 09 — Self-Hosted Visibility

🔑 **The visibility store powers list / filter queries and the Web UI. Standard visibility is gone in 1.24 — run Elasticsearch / OpenSearch, or PG12+/MySQL 8.0.17+ for advanced visibility. See [[Temporal EN 10 - Visibility]].**

## Advanced vs Standard

- **Standard visibility** — legacy, deprecated v1.21, **removed v1.24**. Cassandra-only.
- **Advanced visibility** — current. Supports custom search attributes, complex queries, batch ops via the Web UI. Required for any non-trivial production workload.

## Supported Backends

| Backend | Min server | Notes |
|---|---|---|
| Elasticsearch v7+ | 1.7+ | Recommended for large fleets. |
| Elasticsearch v8+ | 1.18+ | |
| OpenSearch 2+ | 1.30.1+ | Same flow as ES. |
| PostgreSQL 12+ | 1.20+ | Advanced visibility for SQL shops. |
| MySQL 8.0.17+ | 1.20+ | Advanced visibility for SQL shops. |
| SQLite 3.31.0+ | 1.20+ | Dev only; in-memory; no schema upgrade for advanced. |
| Cassandra | — | Legacy standard only — migrate. |

## Elasticsearch / OpenSearch

```yaml
persistence:
  visibilityStore: es-visibility
  datastores:
    es-visibility:
      elasticsearch:
        version: "v7"
        url:
          scheme: "http"
          host: "127.0.0.1:9200"
        indices:
          visibility: temporal_visibility_v1_dev
```

Required ES privileges:

- **Read** index privileges: `create`, `index`, `delete`, `read`.
- **Write** index privilege: `write`.
- **Custom search attributes**: index `manage` privilege + cluster `monitor` or `manage`.

Schema setup uses `temporal-elasticsearch-tool`:

```bash
temporal-elasticsearch-tool \
  --endpoint "$ES_SCHEME://$ES_HOST:$ES_PORT" \
  update-schema --index "$ES_VISIBILITY_INDEX"
```

## PostgreSQL Advanced Visibility

```yaml
persistence:
  visibilityStore: postgres-visibility
  datastores:
    postgres-visibility:
      sql:
        pluginName: 'postgres12'
        databaseName: 'temporal_visibility'
        connectAddr: '127.0.0.0:5432'
        connectProtocol: 'tcp'
        user: 'username_for_auth'
        password: 'password_for_auth'
        maxConns: 2
        maxIdleConns: 2
        maxConnLifetime: '1h'
```

## MySQL Advanced Visibility

```yaml
persistence:
  visibilityStore: mysql-visibility
  datastores:
    mysql-visibility:
      sql:
        pluginName: 'mysql8'
        databaseName: 'temporal_visibility'
        connectAddr: '127.0.0.0:3306'
        connectProtocol: 'tcp'
        user: 'username_for_auth'
        password: 'password_for_auth'
        maxConns: 2
        maxIdleConns: 2
        maxConnLifetime: '1h'
```

Schema:

```bash
temporal-sql-tool --plugin mysql8 \
  --ep $MYSQL_SEEDS -u $MYSQL_USER -p ${DB_PORT:-3306} \
  --db temporal_visibility \
  update-schema -d /etc/temporal/schema/mysql/v8/visibility/versioned
```

## SQLite (Dev)

```yaml
persistence:
  visibilityStore: sqlite-visibility
  datastores:
    sqlite-visibility:
      sql:
        pluginName: 'sqlite'
        databaseName: 'default'
        connectAddr: 'localhost'
        connectProtocol: 'tcp'
        connectAttributes:
          mode: 'memory'
          cache: 'private'
        maxConns: 1
        maxIdleConns: 1
```

⚠️ File-mode SQLite **does not support schema upgrades** for advanced visibility. In-memory only.

## Custom Search Attributes

Per-namespace for SQL backends; cluster-wide for ES:

```bash
temporal operator search-attribute create \
  --name CustomerId \
  --type Keyword \
  --namespace billing-prod
```

Remove:

```bash
temporal operator search-attribute remove \
  --name CustomerId \
  --namespace billing-prod
```

⚠️ **Never put PII, secrets, or sensitive data in search-attribute names or values.** They're stored unencrypted in the visibility store and surfaced in the Web UI. See [[Temporal BP 08 - Security Controls]].

## Dual Visibility (Migration)

Run primary + secondary stores during migration:

```yaml
persistence:
  visibilityStore: cass-visibility
  secondaryVisibilityStore: mysql-visibility
  datastores:
    cass-visibility:
      cassandra:
        hosts: '127.0.0.1'
        keyspace: 'temporal_primary_visibility'
    mysql-visibility:
      sql:
        pluginName: 'mysql8'
        databaseName: 'temporal_secondary_visibility'
```

Dynamic config switches read/write modes:

```yaml
system.secondaryVisibilityWritingMode:
  - value: 'dual'
    constraints: {}
system.enableReadFromSecondaryVisibility:
  - value: false
    constraints: {}
```

Migration sequence:

1. Configure secondary, set writing mode to `dual` — both stores receive writes.
2. Wait for retention period or close all legacy executions.
3. Flip `enableReadFromSecondaryVisibility: true` → reads come from secondary.
4. Promote secondary to primary; drop dual config.

## Dynamic Config Knobs

See [[Temporal RF 07 - Dynamic Configuration]] for ES refresh interval, batch size, and per-namespace search-attribute caps.

⚠️ ES index settings (refresh interval, replicas, shard count) materially affect both write throughput and query freshness. Tune for your scale; defaults are conservative.

💡 **Takeaway:** Run advanced visibility (ES/OpenSearch for scale, PG12+/MySQL 8 for smaller SQL shops). Dual-write during migrations, never store PII in search attributes, and tune ES refresh to match your read-after-write needs.
