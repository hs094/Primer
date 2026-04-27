# SH 11 — Archival and Replication

🔑 **Archival saves closed Workflow histories beyond retention; multi-cluster replication mirrors a Global Namespace to passive clusters for DR. Both are experimental, off-by-default, and require careful per-namespace setup.**

## Part 1 — Archival

### What It Does

Backs up closed Workflow Execution Event Histories (and optionally Visibility records) from primary persistence to blob storage when retention expires. Lets [[Temporal EN 07 - Event History]] outlive retention without overwhelming the persistence store. Useful for compliance, audit, debugging.

⚠️ **Archival is experimental, disabled by default, not supported via Docker Compose images.** Enable explicitly in server config and per namespace.

### Providers

| Provider | Use |
|---|---|
| `filestore` | Local disk. Testing only — APIs do not function with archived histories. |
| `gstorage` | Google Cloud Storage via credentials file. |
| `s3store` | AWS S3 bucket. |
| Custom | Implement `HistoryArchiver` + `VisibilityArchiver`. |

### Service-Level Config

```yaml
archival:
  history:
    state: 'enabled'
    enableRead: true
    provider:
      filestore:
        fileMode: '0666'
        dirMode: '0766'
      gstorage:
        credentialsPath: '/tmp/gcloud/keyfile.json'
```

- `state: enabled` — activates archival for the cluster.
- `enableRead: true` — allows Web UI / CLI to read archived histories.
- `provider:` — configure the providers you'll reference per namespace.

### Namespace Defaults

```yaml
namespaceDefaults:
  archival:
    history:
      state: 'enabled'
      URI: 'file:///tmp/temporal_archival/development'
```

Applied when a namespace is created without explicit archival values.

### Create An Archiving Namespace

```bash
./temporal operator namespace create \
  --namespace my-namespace \
  --global false \
  --history-archival-state enabled \
  --history-archival-uri "s3://my-bucket/temporal/my-namespace" \
  --retention 4d
```

⚠️ **Archival URI is immutable after namespace creation.** Each namespace supports one URI. Different namespaces can use different URIs / providers.

Minimum retention is 1 day; archival is supported on Global Namespaces (replicated multi-cluster).

### Processing Timeline

When a workflow closes, Temporal schedules a close-processing task asynchronously after a randomized delay (up to 5 minutes by default, controlled by `history.archivalProcessorArchiveDelay`). Closed executions remain in primary persistence until retention cleanup runs — both copies coexist briefly.

### Reading Archived Histories

```bash
./temporal workflow show \
  --workflow-id my-workflow-id \
  --run-id my-run-id \
  --namespace my-namespace
```

⚠️ Filestore-archived histories are **not retrievable via APIs** — only inspectable on the host filesystem. Use S3 / GCS for any real workload.

### Custom Archiver

1. New package under `/common/archiver/yourImplementation/`.
2. Implement `HistoryArchiver` + `VisibilityArchiver` interfaces.
3. Update `GetHistoryArchiver` / `GetVisibilityArchiver`.
4. Add config block in `config/development.yaml`.
5. Extend struct in `/common/config.go`.

Use `ArchiverOptions` for retry-progress tracking and to mark non-retryable errors.

## Part 2 — Multi-Cluster Replication

### What It Does

Asynchronously replicates Workflow Executions from an **active** cluster to one or more **passive** clusters for backup / DR. Eventually consistent (not synchronous). Operators can fail over to a passive cluster when needed.

⚠️ **Experimental.** Eventual consistency only — no guarantees of zero data loss at failover.

### Global Namespaces

Replication operates per **Global Namespace**. Enabling adds replication tasks to a parent task queue tied to that namespace.

> All Visibility APIs can be used against active and standby clusters.

Standby reads may show replication lag.

### Cluster Setup (v1.14+)

Each cluster has a unique identity in `clusterMetadata`:

```yaml
clusterMetadata:
  enableGlobalNamespace: true
  failoverVersionIncrement: 100
  masterClusterName: 'cluster-a'
  currentClusterName: 'cluster-a'
  clusterInformation:
    cluster-a:
      enabled: true
      initialFailoverVersion: 1
      rpcAddress: 'cluster-a.frontend:7233'
    cluster-b:
      enabled: true
      initialFailoverVersion: 2
      rpcAddress: 'cluster-b.frontend:7233'
```

Constraints:

- `failoverVersionIncrement` **must be identical** across all clusters in the topology.
- `initialFailoverVersion` **must be unique per cluster** and `< failoverVersionIncrement`.
- All clusters must reach each other on the configured RPC addresses.

### Cluster CRUD via [[Temporal CL 07 - operator]]

```bash
temporal operator cluster upsert --frontend_address "127.0.2.1:8233"
temporal operator cluster upsert --frontend_address "localhost:8233" --enable_connection false
temporal operator cluster remove --name "clusterName"
```

### Versioning & Failover

Each Workflow Execution event carries a version. On failover, the namespace version updates to the smallest `version` such that:

- `version >= old version`
- `version % failoverVersionIncrement == active cluster's initialFailoverVersion`

A cluster may **mutate** an execution only when:

1. `namespaceVersion % failoverVersionIncrement == cluster.initialFailoverVersion`
2. `lastEventVersion <= namespaceVersion`

This keeps writes single-master per execution while allowing read replicas elsewhere.

### Conflict Resolution

History divergence creates branched event trees. The branch with the **highest version** is the `current branch` and drives mutable state rebuilds. Lower-version branches are dead.

### Activity Behavior During Failover

⚠️ **Activity completions are not forwarded across clusters.** Outstanding activities at failover time will time out. They are then re-dispatched to the new active cluster via normal retry logic — design activities to be idempotent.

### Zombie Workflows

Async replication means runs can arrive out of order. A run becomes a "zombie" when its namespace is not active on the local cluster — it cannot be mutated locally. This preserves the "at most one open run per workflow ID" invariant.

### Network Requirements

- All clusters must talk frontend-to-frontend over the cluster RPC address.
- Latency between clusters affects replication lag and failover RTO.
- mTLS recommended on inter-cluster RPC ([[Temporal SH 07 - Security]]).

⚠️ **No RBAC for replication topology** in self-hosted — anyone with `operator cluster upsert` rights can wire a new cluster into your topology. Lock down operator CLI access.

## When To Use Each

| Need | Tool |
|---|---|
| Histories outlive retention | Archival |
| Compliance / audit trail | Archival |
| Cross-region DR | Multi-cluster replication |
| Active-active geo-routing | Multi-cluster replication (with app-side routing) |
| Both | Both — archival is per-namespace; replication is per-cluster topology |

💡 **Takeaway:** Archival = long-term blob backup of closed executions (per-namespace, immutable URI). Replication = async DR mirror of Global Namespaces (eventual consistency, idempotent activities required). Both are experimental and ops-heavy — design failover and recovery runbooks before enabling.
