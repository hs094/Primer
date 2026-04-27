# EN 11 — Temporal Service

🔑 **The Temporal Service = the Server (four roles) + the Persistence store + the Visibility store. It's what the cluster used to be called.**

## Definition

The Temporal Service is the group of services — known collectively as the Temporal Server — combined with a **Persistence** store and a **Visibility** store, that together act as one component of the Temporal Platform.

Naming note: "Temporal Cluster" was renamed to "Temporal Service." Older docs and SDK config keys may still use "cluster."

## Top-Level Components

1. **Temporal Server** — four-role process responsible for orchestrating workflows.
2. **Persistence** — durable storage for event history and mutable state.
3. **Visibility** — indexed projection for listing/filtering executions ([[Temporal EN 10 - Visibility]]).
4. **Archival** (optional) — long-term storage for closed workflows beyond retention.
5. **Multi-Cluster Replication** (optional) — cross-region async replication.

## The Four Server Roles

The Server runs four independently scalable services. In production each is deployed on its own pool.

### Frontend Service

Stateless gRPC gateway exposing the Proto API.
- Rate limits, authorizes, validates, and routes all inbound calls.
- Handles namespace CRUD, external events, worker polls, visibility requests.
- Routes by hashing workflow ID against a hash ring of History service members.
- Default ports: gRPC `7233`, membership `6933`.

### History Service

Owns workflow execution state and event history.
- Work is partitioned across **History Shards**.
- Recommended ratio: ~1 History process per 500 shards.
- Adds tasks to the matching layer as workflows progress.
- Default ports: gRPC `7234`, membership `6934`.

#### History Shards

Shards are the fundamental scaling unit. **The shard count = the number of concurrent database operations the service can perform.**

- Each shard maps 1:1 to a persistence partition.
- **Must be set before first DB write and cannot be changed afterward.** Pick once, pick carefully.
- Practical range: 1 (testing) to 128K (large-scale production).
- Each shard maintains: event history, mutable state, and internal task queues for transfer, timer, replication, and visibility tasks.

⚠️ Under-shard early and you can never grow without a full data migration. Production deployments should plan capacity upfront.

### Matching Service

Hosts user-facing **Task Queues** and dispatches tasks to workers.
- Matches polling workers to pending workflow/activity tasks.
- Scales horizontally — task queues are partitioned across instances.
- Default ports: gRPC `7235`, membership `6935`.

### Worker Service

Internal background processing — replication queues, system workflows (e.g. archival, schedule scavenger).
- Default port: membership `6939`.

⚠️ Don't confuse the **Worker Service** (a server role, runs internally) with **Workers** ([[Temporal EN 06 - Workers]], your application processes). Different things.

## Persistence

Stores event history and mutable workflow state. Backends:
- Cassandra
- MySQL / PostgreSQL
- SQLite (testing only)

Persistence performance is the dominant factor in Temporal Service throughput. The History shard count is fixed against the persistence schema.

## Visibility Store

A separate (or co-located) store for the visibility index. See [[Temporal EN 10 - Visibility]].

## Archival

Optional long-term storage that captures closed workflow event histories beyond the namespace's retention window. Targets pluggable blob stores.

## Multi-Cluster Replication

Optional async replication of namespaces between Temporal Services for HA / DR. A namespace replicated across services has one **active** cluster (accepts writes) and one or more **standby** clusters.

- Self-hosted: configurable per namespace. See [[Temporal SH 01 - Self-Hosted Overview]] and [[Temporal SH 06 - Self-Hosted Namespaces]].
- Cloud: built-in HA replication for Cloud namespaces. See [[Temporal TC 01 - Cloud Overview]].

## Multi-tenancy via Namespaces

A single Temporal Service hosts many [[Temporal EN 12 - Namespaces]]. Namespaces isolate workflow IDs, retention, archival config, and resource impact — heavy traffic in one namespace does not starve another on the same service.

## Deployment Modes

- **Self-hosted** — operate the Server, Persistence, and Visibility stores yourself. See [[Temporal SH 01 - Self-Hosted Overview]] and [[Temporal SH 07 - Security]].
- **Temporal Cloud** — managed Temporal Service. See [[Temporal TC 01 - Cloud Overview]].

## Operator Surface

- CLI: [[Temporal CL 07 - operator]] for namespace, search attribute, cluster admin.
- CLI: [[Temporal CL 12 - workflow]] for workflow operations.

## Cross-references

- Workflows in this service: [[Temporal EN 03 - Workflows]]
- Workers polling matching: [[Temporal EN 06 - Workers]]
- Activities executed by workers: [[Temporal EN 04 - Activities]]
- Event History: [[Temporal EN 07 - Event History]]

💡 **Takeaway:** Temporal Service = Server (Frontend + History + Matching + Worker) + Persistence + Visibility. Pick your History shard count once — it's permanent.
