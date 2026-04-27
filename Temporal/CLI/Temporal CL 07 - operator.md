# CL 07 — operator

🔑 **One-line: cluster-wide admin: Namespaces, Search Attributes, Cluster federation, and Nexus endpoints.**

`temporal operator` is the privileged surface — anything that mutates the [[Temporal EN 11 - Temporal Service]] itself rather than a single Workflow. Four sub-groups: `cluster`, `namespace`, `search-attribute`, `nexus`.

## Subcommand groups

| Group | Purpose |
|---|---|
| `cluster` | Inspect / federate Temporal clusters |
| `namespace` | CRUD [[Temporal EN 12 - Namespaces]] |
| `search-attribute` | Define custom Search Attributes used by `--query` |
| `nexus endpoint` | Register Nexus endpoints for cross-namespace calls |

## cluster

| Sub | Purpose |
|---|---|
| `describe` | Show this cluster's info (`--detail` for verbose) |
| `health` | Frontend health check (good liveness probe) |
| `system` | Server version + diagnostic info |
| `list` | Registered remote clusters (`--limit`) |
| `upsert` | Add/update a remote cluster (`--frontend-address`, `--enable-connection`, `--enable-replication`) |
| `remove` | Deregister a cluster (`--name`) |

```bash
temporal operator cluster health
temporal operator cluster describe --detail
temporal operator cluster upsert \
  --frontend-address dr.tmprl:7233 \
  --enable-connection --enable-replication
```

## namespace

| Sub | Purpose |
|---|---|
| `create` | Register a new namespace |
| `describe` | Inspect by `--namespace` or `--namespace-id` |
| `list` | List all namespaces |
| `update` | Mutate retention, archival, owner, etc. |
| `delete` | Remove (`--yes/-y` to skip confirm) |

Common flags on `create`/`update`:

| Flag | Purpose |
|---|---|
| `--namespace` | Name (required) |
| `--retention` | Workflow history TTL (e.g. `72h`) |
| `--description`, `--email` | Metadata |
| `--global` | Make it Global (multi-cluster) |
| `--active-cluster`, `--cluster` | Active + replica clusters |
| `--data` | Custom KV map (`KEY=VALUE`) |
| `--history-archival-state`, `--history-uri` | Archival sink |
| `--visibility-archival-state`, `--visibility-uri` | Visibility archival |

```bash
temporal operator namespace create \
  --namespace billing \
  --retention 168h \
  --description "Billing team workflows"

temporal operator namespace update \
  --namespace billing --retention 720h

temporal operator namespace delete --namespace stale-ns -y
```

## search-attribute

Custom [[Temporal DV 02 - Workflow Basics|Search Attributes]] used by `--query` filters across CLI commands.

| Sub | Purpose |
|---|---|
| `create` | Add custom attribute (`--name`, `--type`) |
| `list` | List active attributes |
| `remove` | Drop attribute (`--name`, `-y`) |

Supported `--type` values: `Text`, `Keyword`, `Int`, `Double`, `Bool`, `Datetime`, `KeywordList`.

```bash
temporal operator search-attribute create \
  --name CustomerId --type Keyword

temporal operator search-attribute create \
  --name OrderTotal --type Double

temporal operator search-attribute list
temporal operator search-attribute remove --name OrderTotal -y
```

## nexus endpoint

Register cross-namespace Nexus endpoints (for the Nexus service-handler pattern).

| Sub | Purpose |
|---|---|
| `create` | Register endpoint (`--name`, `--target-namespace`, `--target-task-queue`, `--target-url`, `--description`/`--description-file`) |
| `get` | Retrieve by `--name` |
| `list` | List endpoints |
| `update` | Mutate target/description; `--unset-description` clears |
| `delete` | Remove by `--name` |

```bash
temporal operator nexus endpoint create \
  --name payments-svc \
  --target-namespace payments-prod \
  --target-task-queue nexus-handler \
  --description "Charge / refund operations"

temporal operator nexus endpoint list
```

## Common patterns

```bash
# New project bootstrap
temporal operator namespace create --namespace orders --retention 168h
temporal operator search-attribute create --name OrderId --type Keyword
temporal operator search-attribute create --name CustomerTier --type Keyword

# Liveness check from a script
temporal operator cluster health || exit 1
```

⚠️ `namespace delete` does **not** preserve workflow histories — they're tombstoned and unreachable. Confirm with `describe` before deletion; `-y` skips the safety prompt.

⚠️ Search attributes are **cluster-wide**, not per-namespace. Plan names carefully (`OrderId`, not `Id`); removing one breaks every Workflow that references it in `--query`.

⚠️ `cluster upsert --enable-replication` only does anything if both clusters are configured for Global Namespaces — otherwise it's a no-op stub.

💡 **Takeaway:** `operator` is the "DBA" surface — namespaces, search attribute schema, cluster federation, Nexus wiring. Use sparingly, audit access.
