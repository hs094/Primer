# EN 10 — Visibility

🔑 **Visibility is the subsystem that lets operators list, filter, and search Workflow Executions across the service.**

## Definition

"Visibility" refers to the subsystems and APIs that enable an operator to view, filter, and search for Workflow Executions that currently exist (or recently existed) within a Temporal Service. It powers the Web UI, the `tctl`/`temporal` CLI, and any List/Count API call.

## Visibility Store

Visibility data lives in a **Visibility Store**, which is part of the Persistence layer. It indexes a denormalized projection of workflow execution events so that listing, filtering, and counting are efficient.

Supported backends:
- **SQL** — MySQL 8.0.17+, PostgreSQL 12+ (from Server v1.20).
- **Elasticsearch** — v7 with Server v1.7+, v8 with Server v1.18+.
- **OpenSearch** — 2+ with Server v1.30.1+.
- **Temporal Cloud** — enabled by default; managed for you.

## Dual Visibility

Since Server v1.21, a service can run with **two** visibility stores simultaneously (Dual Visibility). This exists to migrate from one backend to another (e.g. SQL to Elasticsearch) without downtime: writes go to both, reads come from the primary.

## Search Attributes

Visibility queries filter by **Search Attributes** — typed key/value pairs indexed against each workflow execution.

**Default (system) Search Attributes** include:
- `WorkflowType`
- `WorkflowId`
- `RunId`
- `ExecutionStatus`
- `StartTime`, `CloseTime`, `ExecutionTime`
- `TaskQueue`

**Custom Search Attributes** are user-defined, registered per-namespace, and typed (Keyword, Text, Int, Double, Bool, Datetime, KeywordList). Workflow code calls `upsert_search_attributes` to set them as state evolves.

⚠️ Search Attributes are **eventually consistent**. A workflow that just upserted an attribute may not appear in a List query for a short window.

## List Filter

Listing uses a SQL-like query language called a **List Filter**:

```sql
WorkflowType = "OrderWorkflow"
  AND ExecutionStatus = "Running"
  AND StartTime > "2025-01-01T00:00:00Z"
  ORDER BY StartTime DESC
```

Supported clauses: `=`, `!=`, `>`, `<`, `>=`, `<=`, `IN`, `BETWEEN`, `STARTS_WITH`, `IS NULL`, combined with `AND` / `OR` / parentheses, and `ORDER BY`.

## Count API

The Count API returns the number of executions matching a filter. It supports:

```sql
SELECT COUNT(*) FROM ...
GROUP BY ExecutionStatus
```

`GROUP BY ExecutionStatus` is currently the only supported grouping. Counts are **approximate** — the system trades exactness for speed at scale.

## Legacy: Standard vs. Advanced Visibility

Older docs split "standard visibility" (basic filters on a few system attributes) from "advanced visibility" (full List Filter against a search-aware backend). Standard Visibility was deprecated in v1.21 and **removed** in v1.24. Modern services run what was previously called advanced visibility.

⚠️ If you read older runbooks referring to `--enable-advanced-visibility`, that flag and its surrounding plumbing are gone.

## Operational Surface

- CLI: [[Temporal CL 07 - operator]] handles namespace + search attribute registration.
- CLI: [[Temporal CL 12 - workflow]] uses List Filters in `workflow list` / `workflow count`.
- Cloud namespaces ship visibility on Elasticsearch automatically — see [[Temporal TC 03 - Cloud Namespaces]].
- Self-hosted: see [[Temporal SH 01 - Self-Hosted Overview]] for visibility store deployment.

## Cross-references

- Service architecture context: [[Temporal EN 11 - Temporal Service]]
- Namespace scoping for search attributes: [[Temporal EN 12 - Namespaces]]

💡 **Takeaway:** Visibility = the index over your workflows. Register custom Search Attributes per namespace, write them via `upsert_search_attributes`, and treat results as eventually consistent.
