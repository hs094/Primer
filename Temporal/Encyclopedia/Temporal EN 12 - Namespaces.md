# EN 12 — Namespaces

🔑 **A Namespace is the unit of isolation inside a Temporal Service — it scopes Workflow IDs, retention, archival, and resource impact.**

## Definition

A Namespace is a unit of isolation within the Temporal Platform. Every Workflow Execution and every Task Queue belongs to exactly one Namespace. A single Temporal Service hosts many Namespaces.

## What a Namespace Scopes

### Workflow ID Uniqueness

Workflow IDs are unique **within** a Namespace. The same ID can exist across different Namespaces with no conflict — Namespace + Workflow ID + Run ID is the global identity.

### Resource Isolation

Heavy traffic in one Namespace does not starve other Namespaces on the same Temporal Service. The service applies per-namespace rate limits and quotas.

### Configuration Boundary

Per-Namespace, not global, configuration includes:
- **Retention Period** — how long closed workflow histories are kept before deletion.
- **Archival** destination and config (long-term storage past retention).
- **Search Attribute** registrations (custom keys).
- **Replication** config in multi-cluster setups.

### Multi-cluster Replication

A Namespace can be replicated across Temporal Services. One cluster is **active** (accepts writes), others are **standby**. Failover changes which cluster is active for that namespace.

## The "default" Namespace

When no Namespace is supplied, SDKs and the CLI use the literal name `default`. **A Namespace must be created before use** — `default` is not auto-created on every server build.

⚠️ Forgetting `--namespace` (or the SDK config) and silently writing to `default` is a common foot-gun in shared environments.

## Multi-Tenancy

A single Namespace is itself multi-tenant — multiple apps or teams can share it. But they must coordinate naming conventions for Workflow IDs and Task Queues to avoid collisions.

For stronger isolation (rate limits, retention, RBAC), give each tenant or environment its own Namespace.

## Naming

Common patterns:
- `<service>-<env>` — e.g. `payments-prod`, `payments-staging`.
- `<team>` — coarser, when teams own multiple services.

⚠️ Namespace names are stable and operationally meaningful — renaming is non-trivial. Pick a convention before you scale.

## Retention Period

Retention defines how long the system keeps a closed Workflow Execution's event history. After retention expires, the execution is deleted from primary storage (and, if Archival is configured, archived first).

- Self-hosted: defaults vary; configurable per namespace.
- Cloud: bounded range, set per Cloud namespace.

⚠️ A retention period that is too short can break debugging and audit. Too long inflates persistence cost. Tune per workload.

## Self-hosted vs. Cloud Namespaces

### Self-hosted

You create and manage Namespaces via the CLI ([[Temporal CL 07 - operator]]) or admin API. You control the persistence/visibility store, replication, archival, and any auth integration. See [[Temporal SH 06 - Self-Hosted Namespaces]] and [[Temporal SH 07 - Security]].

### Temporal Cloud

Cloud Namespaces are managed and ship with extras:
- API key and mTLS authentication out of the box.
- Built-in role-based access control (RBAC).
- High-availability replication.
- Namespace tagging for org-level grouping.

See [[Temporal TC 03 - Cloud Namespaces]] and [[Temporal TC 01 - Cloud Overview]].

## Operator Surface

```bash
temporal operator namespace create --namespace payments-prod
temporal operator namespace describe --namespace payments-prod
temporal operator namespace update --namespace payments-prod \
  --retention 30d
```

## Cross-references

- Hosting service: [[Temporal EN 11 - Temporal Service]]
- Visibility scoped per namespace: [[Temporal EN 10 - Visibility]]
- Cross-namespace coordination: [[Temporal EN 13 - Temporal Nexus]]
- Workflows live inside a namespace: [[Temporal EN 03 - Workflows]]

💡 **Takeaway:** Use a Namespace per tenant / env / strong-isolation boundary. They scope Workflow ID uniqueness, retention, archival, and resource impact.
