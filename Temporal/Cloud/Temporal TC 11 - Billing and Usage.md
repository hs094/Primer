# TC 11 — Billing and Usage

🔑 **Cloud bills on Actions plus storage; the Billing Center, Billing API, Usage Dashboards, and Actions metrics tell you where the money goes — at namespace, project, and minute granularity.**

## What Cloud charges for
Usage is measured along two axes:

- **Actions** — the metered events Temporal counts (Workflow start, Activity completion, Signal/Query, Timer fire, etc.). Actions are the primary billable unit.
- **Storage** — Event History bytes retained per namespace, scaled by [[Temporal EN 12 - Namespaces]] retention period.
- **Services and support** — plan-tier add-ons (Business, Enterprise, Mission Critical), [[Temporal TC 13 - High Availability]] replication, [[Temporal TC 09 - SAML and SCIM]] features.

The public Pricing page carries current rates; the platform itself doesn't expose them — focus your tooling on **measuring** usage and let pricing live in the contract.

## Where to see usage

### Billing Center
The UI for **Account Owners** and **Finance Admins**. Lists invoices, remaining credits, plan details, and lets you change plans or update payment information. This is the only surface that exposes invoice-level data.

### Billing API (Pre-release)
Programmatic access to billing data on a **per-Namespace** basis at **hourly** granularity, enriched with Tags and Projects. Use this for chargeback, FinOps automation, or piping costs into your own data warehouse. Pre-release means the schema may shift.

### Usage Dashboards
In-UI namespace-level Action aggregations. Visible to **Global Admins, Namespace Admins, and Developers** at varying granularity — Developers see only their accessible namespaces.

### Actions metrics (Public Preview)
High-cardinality, **minute-level** billable Action data exposed through the [[Temporal TC 10 - Cloud Metrics]] endpoint. Requires a Service Account with the **Metrics Read-Only** role (see [[Temporal TC 08 - Users and Groups]]). Best for live alerting on cost spikes.

## Role visibility matrix

| Surface | Account Owner | Finance Admin | Global Admin | Namespace Admin | Developer |
|---|---|---|---|---|---|
| Billing Center / invoices | yes | yes | no | no | no |
| Billing API | yes | yes | partial | no | no |
| Usage Dashboards | yes | yes | yes | own ns | own ns |
| Actions metrics | yes | yes | yes | own ns | own ns |

(Approximate — confirm in the docs when granting roles.)

## What drives the bill
A short list of the usual suspects:

- **Hot Activities** — chatty integrations or per-event `execute_activity` calls explode Action counts. Batch where possible.
- **Long retention** — keeping closed Workflow histories beyond what your audit window requires. Trim retention per-namespace.
- **Heavy Schedules / cron** — every action a Schedule kicks off is billed. See [[Temporal DV 08 - Schedules]].
- **Heartbeats and Timers** — high-frequency heartbeats and short Timers add up.
- **Replication** — HA namespaces double-write across regions/clouds; budget the multiplier.

⚠️ Action metering is **per-namespace**, but rate limits are **per-account**. A noisy non-prod namespace can starve prod even before the invoice arrives — split projects into namespaces and watch each one.

## Cost optimization patterns
- Group fine-grained work into a single Activity rather than dispatching many.
- Use [[Temporal DV 04 - Continue As New]] to cap history size on long-running Workflows.
- Set Namespace retention to the smallest period your runbooks tolerate.
- Use SDK-side caching/batching to drop repeated Queries and Signals.
- Tag namespaces with cost-center labels and use the Billing API to allocate spend.
- Audit Schedules and Workflow Search Attribute usage in [[Temporal EN 10 - Visibility]] queries.

See [[Temporal BP 09 - Cost Optimization]] for a fuller playbook.

## Notifications
Cloud emails Account Owners and Finance Admins when:

- Sign-up credits reach 50% / 90% consumption.
- Sign-up credits expire — alerts at 30, 14, 7, and 1 day.
- Plan type changes.

See [[Temporal TC 14 - Cloud Operations and tcld]] for managing notification preferences.

## Tags and Projects
The Billing API enriches per-namespace data with **Tags** and **Projects** so you can map spend back to teams, environments, or customer tenants. Apply tags at namespace creation time (via `tcld`, Terraform, or the UI) and treat them as the chargeback key:

- One namespace per environment per service is a clean default.
- Tag with `cost_center`, `env`, `team` — whatever your finance org reports against.
- Pull the Billing API into your warehouse hourly; reconcile against invoices monthly.

## Plans and credits
Cloud sells in tiered plans (Starter, Business, Enterprise, Mission Critical) — features like [[Temporal TC 09 - SAML and SCIM]], [[Temporal TC 13 - High Availability]], and elevated rate limits gate at higher tiers. New accounts often receive sign-up credits with a fixed expiry; once they burn down or expire, normal billing kicks in.

If you're nearing limits or want to compare plans, the Billing Center surfaces the current plan and a "Change Plan" path; for negotiated arrangements work with your account team.

## What the Billing API surface looks like
Per-namespace, per-hour records typically carry:

- Action counts broken down by action type.
- Storage bytes for active and closed Workflow histories.
- Tags and Project assignments.
- Plan-tier add-on usage where applicable.

Pre-release means the schema is still moving — pin to the documented version header and version your warehouse views accordingly.

## Operational checklist
- Audit retention settings every quarter; closed-history retention is the silent line item.
- Alert on Action-rate spikes through [[Temporal TC 10 - Cloud Metrics]] before invoice surprises.
- Tag every namespace at creation — backfilling tags is harder than it looks.
- Keep at least one Finance Admin distinct from your Account Owner so billing doesn't depend on one human.

💡 **Takeaway:** Track Actions and storage continuously, not just at month-end. Wire Actions metrics into the same dashboards you use for [[Temporal TC 10 - Cloud Metrics]], split workloads into namespaces so you can attribute cost, and treat retention and Activity granularity as the two biggest design knobs you control.
