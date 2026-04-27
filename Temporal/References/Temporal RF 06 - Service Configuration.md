# RF 06 — Service Configuration

🔑 **Static YAML config that boots a Temporal Cluster — what each top-level section controls and the keys you'll actually edit when running [[Temporal SH 01 - Self-Hosted Overview|self-hosted Temporal]].**

This is `config/development.yaml`-style configuration loaded at process startup. For values that change at runtime without a restart, see [[Temporal RF 07 - Dynamic Configuration]]. For sensible production defaults, see [[Temporal SH 04 - Configurable Defaults]].

## `global`

Cross-cutting concerns shared by every service role.

### `global.membership`

| Key | Description | Default |
|---|---|---|
| `maxJoinDuration` | Time before service fails joining the gossip layer. | `10s` |
| `broadcastAddress` | IP for gossip protocol communication. | (host IP) |

### `global.metrics`

| Key | Description |
|---|---|
| `prefix` | Applied to all outgoing metrics. |
| `tags` | Key-value pairs reported with every metric. |
| `excludeTags` | Excludes tags with unbounded cardinality. |
| `statsd` | `hostPort`, `prefix`, `flushInterval` (300ms), `flushBytes` (1432). |
| `prometheus` | `framework` (`tally`/`opentelemetry`), `listenAddress`, `handlerPath` (`/metrics`). |
| `m3` | `hostPort`, `service`, `queue` (4k), `packetSize` (32k). |

### `global.pprof`

| Key | Description |
|---|---|
| `port` | Port for Go pprof profiler. |

### `global.tls`

`internode` and `frontend` subsections, each with server / client TLS configuration. See `tls` block under datastores for shape.

## `persistence`

The database tier. Most production tuning happens here.

| Key | Description |
|---|---|
| `numHistoryShards` | Number of shards. **Immutable after first run** — pick carefully. |
| `defaultStore` | Primary datastore name. |
| `visibilityStore` | Primary Visibility datastore. |
| `secondaryVisibilityStore` | Secondary Visibility datastore (optional, dual-write). |
| `datastores` | Map of named store definitions. |

### `persistence.datastores.<name>` — Cassandra

`hosts`, `port` (9042), `user`, `password`, `keyspace`, `datacenter`, `maxConns`, `tls`.

### `persistence.datastores.<name>` — SQL

`user`, `password`, `pluginName` (`mysql` / `postgres`), `databaseName`, `connectAddr`, `connectProtocol` (`tcp`/`unix`), `connectAttributes`, `maxConns`, `maxIdleConns`, `maxConnLifetime`, `tls`.

### `tls` block (datastore)

`enabled`, `serverName`, `certFile`, `keyFile`, `caFile`, `enableHostVerification`.

## `log`

| Key | Description |
|---|---|
| `stdout` | Output to standard output. |
| `level` | `debug` / `info` / `warn` / `error` / `fatal`. |
| `outputFile` | Log file path. |

## `clusterMetadata`

Multi-cluster / replication settings.

| Key | Description |
|---|---|
| `enableGlobalNamespace` | Enable cross-cluster namespaces. |
| `failoverVersionIncrement` | Version increment on failover. |
| `masterClusterName` | Master cluster designation. |
| `currentClusterName` | Current cluster name. **Immutable**. |
| `replicationConsumer` | `kafka` or `rpc` consumption method. |
| `clusterInformation` | Per-cluster settings: `enabled`, `initialFailoverVersion`, `rpcAddress`. |

## `services`

Per-role configuration. Roles: `frontend`, `matching`, `worker`, `history`.

### `services.<role>.rpc`

| Key | Description |
|---|---|
| `grpcPort` | Port for gRPC service. |
| `membershipPort` | Port for ringpop membership. |
| `bindOnLocalHost` | Bind to localhost only. |
| `bindOnIP` | Specific IP to bind to. |

## `publicClient`

| Key | Description |
|---|---|
| `hostPort` | Frontend connection address. |

## `archival`

Optional history / Visibility archival to long-term storage.

| Key | Description |
|---|---|
| `history.state` / `visibility.state` | `enabled` / `disabled`. |
| `history.enableRead` / `visibility.enableRead` | Whether reads from archive are served. |
| `history.provider` / `visibility.provider` | `filestore`, `gstorage`, or `s3`. |

## `namespaceDefaults`

| Key | Description |
|---|---|
| `archival.history.state` / `archival.history.URI` | Default archival for new namespaces. |
| `archival.visibility.state` / `archival.visibility.URI` | Default visibility archival. |

## `dcRedirectionPolicy`

| Key | Description |
|---|---|
| `policy` | `noop` / `selected-apis-forwarding` / `all-apis-forwarding`. |

## `dynamicConfigClient`

Hooks the static config to the dynamic config file.

| Key | Description |
|---|---|
| `filepath` | Dynamic configuration YAML location. |
| `pollInterval` | Check interval (minimum **5s**). |

⚠️ Gotchas:
- **`numHistoryShards` is immutable** after first run. Choose with future scale in mind — typical production: 512 or 1024+.
- **`currentClusterName` is also immutable** — if you set it wrong, you must wipe state and restart.
- If you set `dcRedirectionPolicy` to anything other than `noop`, requests can be silently forwarded to other clusters in a multi-cluster setup.
- `archival` is opt-in and adds storage cost — don't enable it on every namespace by default unless you've sized the bucket.
- `prometheus.framework`: pick `opentelemetry` for new deployments; `tally` is the legacy option.
- `dynamicConfigClient.pollInterval` minimum is 5s — setting it lower silently bumps to 5s.

💡 **Takeaway:** Static config sets the cluster's identity (shards, name, DBs, TLS); everything you tune at runtime lives in [[Temporal RF 07 - Dynamic Configuration]].
