# SH 01 — Self-Hosted Overview

🔑 **Self-hosting means you own the [[Temporal EN 11 - Temporal Service]] — frontend, history, matching, worker, persistence, visibility — and every ops concern that comes with it.**

## When Self-Host

Open-source Temporal Service deploys in your own infra (cloud VMs, k8s, bare metal). You orchestrate durable applications without depending on [[Temporal TC 01 - Cloud Overview]].

Trade-off: full control + zero per-action cost vs you handle scale, security, upgrades, backup, monitoring, multi-cluster.

## Start Local, Move to Production

Always begin with the Temporal CLI dev server (single binary, SQLite, no deps):

```bash
temporal server start-dev
```

Use it for development regardless of production plans. See [[Temporal CL 09 - server]].

For real workloads, choose deployment style — see [[Temporal SH 02 - Deployment]].

## Components You're Operating

The [[Temporal EN 11 - Temporal Service]] is four services + persistence + visibility:

```
client ─► frontend ──► history ──► persistence (Cassandra/SQL)
              │           ▲
              ├─► matching┤
              └─► worker (system workflows)
                    └─► visibility store (ES / SQL)
```

- **frontend** — gRPC entry point on `:7233`, auth, rate limiting.
- **history** — owns workflow state, sharded by `NumHistoryShards` (build-time).
- **matching** — task queue dispatcher; sync-matches tasks to workers.
- **worker** — runs Temporal's own internal system workflows.
- **persistence** — Cassandra / MySQL / PostgreSQL for state + events.
- **visibility** — Elasticsearch / OpenSearch / SQL for list & filter queries.

## Surface Area You Own

| Concern | Note |
|---|---|
| Topology, persistence, k8s charts | [[Temporal SH 02 - Deployment]] |
| Embedded Go server for tests | [[Temporal SH 03 - Embedded Server]] |
| Limits and tunables | [[Temporal SH 04 - Configurable Defaults]] |
| Production readiness | [[Temporal SH 05 - Production Checklist]] |
| Namespace lifecycle | [[Temporal SH 06 - Self-Hosted Namespaces]] |
| TLS, auth, codecs | [[Temporal SH 07 - Security]] |
| Metrics & dashboards | [[Temporal SH 08 - Monitoring]] |
| ES / SQL visibility store | [[Temporal SH 09 - Self-Hosted Visibility]] |
| Server version bumps | [[Temporal SH 10 - Upgrade Server]] |
| Archival + DR replication | [[Temporal SH 11 - Archival and Replication]] |

## Operational Doctrine

- Treat Temporal hosts like databases: **never expose to the public internet**.
- Service config is YAML + dynamic config — see [[Temporal RF 06 - Service Configuration]] and [[Temporal RF 07 - Dynamic Configuration]].
- Cluster scale is set at *build time*: `NumHistoryShards` cannot be changed later. Pick generously (4096 for serious fleets).
- Sequential upgrades only: `v1.n` → `v1.n+1`, never skip minor versions. See [[Temporal SH 10 - Upgrade Server]].
- Monitor [[Temporal RF 03 - Events]] retention, persistence latency, sync match rate.
- Use [[Temporal CL 07 - operator]] for cluster admin (namespaces, search attributes, system info).

⚠️ **Self-hosted has no built-in RBAC, audit log, or multi-tenant control plane.** You build it (or use [[Temporal TC 01 - Cloud Overview]]). See [[Temporal BP 04 - Multi-Tenant Patterns]] and [[Temporal BP 08 - Security Controls]].

⚠️ **Default `noopAuthorizer` allows every request.** Configure a real `Authorizer` + `ClaimMapper` before any non-ops client connects. See [[Temporal SH 07 - Security]].

⚠️ Cassandra visibility (legacy standard visibility) is **removed in 1.24**. Plan migration to ES or SQL advanced visibility before upgrading. See [[Temporal SH 09 - Self-Hosted Visibility]].

## Self-Hosted vs Cloud

| Concern | Self-Hosted | [[Temporal TC 01 - Cloud Overview|Cloud]] |
|---|---|---|
| Provisioning | You | Anthropic-style managed |
| mTLS / TLS | You configure | Managed |
| RBAC / audit | You build | Built in |
| Schema upgrades | You run schema tools | Managed |
| Shard sizing | Build-time decision (irreversible) | Managed |
| Multi-cluster DR | You wire & test | Available as feature |
| Cost shape | DB + ES + ops salary | Per-action billing |

## Path Forward

1. Run `temporal server start-dev` locally; build workflows.
2. Pick deployment shape (Docker Compose, Helm, binaries) — [[Temporal SH 02 - Deployment]].
3. Walk [[Temporal SH 05 - Production Checklist]] before sending real traffic.
4. Wire [[Temporal SH 07 - Security]] + [[Temporal SH 08 - Monitoring]] before any public client connects.
5. Plan namespace-per-tenant + retention policy — [[Temporal SH 06 - Self-Hosted Namespaces]].
6. Decide archival + DR posture — [[Temporal SH 11 - Archival and Replication]].

💡 **Takeaway:** Self-hosted Temporal is a database-class system — expect to own provisioning, mTLS, schema upgrades, shard sizing, and DR. If that's not your job, use Temporal Cloud.
