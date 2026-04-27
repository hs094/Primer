# TC 01 — Cloud Overview

🔑 **Temporal Cloud is the managed control plane: you bring workers, they run the Temporal Service, RBAC, and observability.**

## What Temporal Cloud Is

A multi-tenant managed [[Temporal EN 11 - Temporal Service]] across AWS and GCP regions. You get a hosted frontend/history/matching/persistence stack; you keep ownership of Workers, Workflow code, and the Task Queues they poll. No persistence to manage, no Cassandra to babysit — contrast with [[Temporal SH 01 - Self-Hosted Overview]].

The unit of isolation is the **Namespace** (see [[Temporal TC 03 - Cloud Namespaces]]) — same concept as [[Temporal EN 12 - Namespaces]] in self-hosted, but each one carries its own gRPC endpoint, retention, auth config, and RBAC scope.

## Surface Area

Cloud splits cleanly into three planes you configure independently.

**Identity / access control** — see [[Temporal TC 06 - Account Access]]:
- Users (humans, MFA, SAML SSO)
- User Groups (group-based namespace permissions)
- Service Accounts (machine identities for workers/CI)
- Account roles + Namespace permissions (RBAC, see [[Temporal TC 07 - Roles and Permissions]])
- SAML + SCIM (see [[Temporal TC 09 - SAML and SCIM]])

**Authentication for connections**:
- API keys — bearer tokens, see [[Temporal TC 04 - API Keys]]
- mTLS — client certs against your CA bundle, see [[Temporal TC 05 - mTLS Certificates]]
- Each Namespace picks one (mTLS *or* API key) at create time

**Observability + ops**:
- Prometheus metrics endpoint per account ([[Temporal TC 10 - Cloud Metrics]])
- Audit log sink → S3 / GCS
- Billing + usage dashboards ([[Temporal TC 11 - Billing and Usage]])
- Export for archival workflow histories
- Worker / service health views

## Tooling

Three control planes, all backed by the same API:

```bash
# tcld — admin CLI for namespaces, certs, keys, users
tcld login
tcld namespace list

# temporal CLI — same as self-hosted, points at cloud endpoint
temporal workflow list \
  --address my-ns.abc12.tmprl.cloud:7233 \
  --namespace my-ns.abc12 \
  --api-key "$TEMPORAL_API_KEY"
```

- **Web UI** — `cloud.temporal.io` for namespaces, users, billing, workflow inspection
- **`tcld`** CLI — scriptable admin (see [[Temporal TC 14 - Cloud Operations and tcld]])
- **Cloud Ops API** — gRPC API behind everything; underpins Terraform provider
- **Terraform provider** — `temporalio/temporalcloud` for IaC

## Regions and Connectivity

Multi-region across AWS and GCP. Each Namespace is pinned to one region. Workers connect via TLS 1.3 over public internet by default; PrivateLink (AWS) and Private Service Connect (GCP) available — see [[Temporal TC 12 - Connectivity]].

For DR/HA — replicas, RPO/RTO targets, multi-region active/active — see [[Temporal TC 13 - High Availability]].

## Pricing Shape

You pay for **actions** (state transitions: workflow start, activity complete, signal, timer fire, etc.) plus **active storage** (event history retained inside retention window) plus **retained storage** (anything past retention window, archived). Support tier and add-ons (HA, audit log) are line items on top.

⚠️ **Action counts balloon with chatty workflows.** Heartbeating activities every 1s, signals in tight loops, or timers used for polling all bill per emit. Profile with [[Temporal TC 10 - Cloud Metrics]] before scaling.

⚠️ **Email = account.** One email maps to exactly one Cloud account. Use group aliases (`temporal-admins@`) for the founding Account Owner, not someone's personal address — see [[Temporal BP 07 - Cloud Access Control]].

⚠️ **Retention is per-Namespace and capped at 90 days.** Past that, history is gone unless you've configured Export. Don't model long-running audit needs around event history alone.

## Versus Self-Hosted

| Concern | Cloud | Self-Hosted |
|---|---|---|
| Persistence | Managed | You run Cassandra/Postgres/MySQL |
| Upgrades | Continuous, automatic | You schedule, you test |
| RBAC | Built-in (account + namespace) | DIY via authorizer plugin |
| Metrics | Prometheus endpoint | Scrape your own service |
| mTLS | First-class | DIY via TLS config |
| SLA | Contractual (99.9%+ tiers) | Yours |
| Compliance | SOC 2, HIPAA, PCI, ISO 27001 | Yours to attest |

Security posture comparison: [[Temporal SH 07 - Security]] vs [[Temporal BP 08 - Security Controls]].

## What You Still Own

Cloud does **not** run your Workers. That's the whole point — your code, your secrets, your VPC. Cloud only sees Workflow state transitions and serialized Payloads (which you can encrypt end-to-end with a Codec Server, declared per Namespace).

💡 **Takeaway:** Cloud handles the Temporal Service, identity, and observability — you keep workers, code, and payload encryption. Pick Namespace boundaries early; everything else (RBAC, certs, metrics) hangs off them.
