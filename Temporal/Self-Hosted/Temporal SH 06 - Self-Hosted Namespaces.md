# SH 06 — Self-Hosted Namespaces

🔑 **A [[Temporal EN 12 - Namespaces|Namespace]] is the unit of isolation; in self-hosted you create, retain, archive, and authorize them yourself.**

## Register Before You Run

You must create at least one namespace before running workflows. Local dev (`temporal server start-dev`) auto-provisions `default`. Other deployments don't.

```bash
temporal operator namespace create --namespace default
```

⚠️ **Registration takes up to 15 seconds.** Don't immediately call the namespace from a worker — sleep / poll `describe` until it returns.

## Retention Is Required

Closed Workflow Execution histories are kept for the namespace's retention period, then deleted. Pick per environment:

```bash
temporal operator namespace create \
  --namespace billing-prod \
  --retention 30d
```

Minimum retention is 1 day. After retention, history is gone unless [[Temporal SH 11 - Archival and Replication|Archival]] is on. See [[Temporal EN 07 - Event History]].

## CRUD via [[Temporal CL 07 - operator]]

| Op | Command |
|---|---|
| Create | `temporal operator namespace create --namespace X --retention 7d` |
| List | `temporal operator namespace list` |
| Describe | `temporal operator namespace describe --namespace X` |
| Update | `temporal operator namespace update --namespace X --retention 14d` |
| Delete | `temporal operator namespace delete --namespace X` |

You can also create namespaces from the SDKs (`RegisterNamespace` in Go / Java / TypeScript) — useful for bootstrap scripts.

## Deprecate vs Delete

| Action | Effect |
|---|---|
| **Deprecate** | Block new workflow starts; existing executions complete normally. Reversible. |
| **Delete** | Remove namespace + all executions. **Irreversible. No undo.** |

⚠️ **`namespace delete` wipes every workflow execution and history in that namespace.** Use deprecate first; delete only after you've verified you want the data gone (or archived).

## Per-Namespace Archival

Enable history archival at namespace creation; URI is **immutable** afterward:

```bash
temporal operator namespace create \
  --namespace billing-prod \
  --retention 30d \
  --history-archival-state enabled \
  --history-archival-uri "s3://my-bucket/temporal/billing-prod"
```

See [[Temporal SH 11 - Archival and Replication]].

## Namespace-Default Config

Server-level defaults applied to namespaces created without explicit values:

```yaml
namespaceDefaults:
  archival:
    history:
      state: 'enabled'
      URI: 'file:///tmp/temporal_archival/development'
```

## Custom Search Attributes

Self-hosted attaches search attributes to a specific namespace:

```bash
temporal operator search-attribute create \
  --name CustomerId \
  --type Keyword \
  --namespace billing-prod
```

ES-backed visibility is cluster-wide; SQL advanced visibility (PG12+/MySQL 8.0.17+) keeps attributes per-namespace. See [[Temporal SH 09 - Self-Hosted Visibility]].

## Authorization

⚠️ **Without a configured `Authorizer`, the default is `noopAuthorizer` — every namespace operation is allowed.** Mount a custom `Authorizer` + `ClaimMapper` on the frontend before exposing to anyone outside the ops team. See [[Temporal SH 07 - Security]] and [[Temporal BP 08 - Security Controls]].

## Multi-Tenant Pattern

For multi-tenant fleets, a namespace-per-tenant gives clean isolation, retention, and quota boundaries. See [[Temporal BP 04 - Multi-Tenant Patterns]].

💡 **Takeaway:** Create namespaces with explicit retention and archival; never rely on the default `noopAuthorizer` in production; namespace delete is an irreversible blast radius.
