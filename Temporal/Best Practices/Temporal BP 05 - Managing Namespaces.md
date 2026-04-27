# BP 05 — Managing Namespaces

🔑 **Treat namespace name as an architectural decision: lowercase, hyphenated, `<use-case>-<domain>-<env>` — and isolate one use-case-environment per namespace.**

## Naming Conventions

### Lowercase + Hyphens
> "Use lowercase letters and hyphens (`-`) as separators in Namespace names."

This avoids confusion between case-sensitive (open source) and case-insensitive (Cloud) environments. Example: `payment-checkout-prd`.

### Consistent Pattern
Follow `<use-case>-<domain>-<environment>` with rough length budgets:

| Component | Max chars | Examples |
|---|---|---|
| use case | 10 | `payments`, `orders` |
| domain | 10 | `checkout`, `inventory` |
| environment | 3 | `dev`, `stg`, `prd` |

This pattern "allows platform teams to implement chargeback to application teams" and isolates namespace-level limits cleanly.

## Organizational Patterns

### Pattern 1 — Use Case + Environment
`<use-case>_<environment>` → `payments_prod`, `orders_dev`. For simple setups without multiple services or team boundaries.

### Pattern 2 — Use Case + Service + Environment
`<use-case>_<service>_<environment>` → `payments_gateway_prod`. For multiple services that talk externally via HTTP/gRPC inside the same use case.

### Pattern 3 — Use Case + Domain + Environment
`<use-case>_<domain>_<environment>` → `payments_checkout_prod`. For systems where services need internal communication. **Use Temporal Nexus to connect Workflows across namespace boundaries** for improved security and isolation.

## Workflow ID Uniqueness

When multiple teams share a namespace:
- Prefix Workflow IDs with **service-specific strings**.
- Keep **Task Queue names unique** within the namespace.

```text
workflow_id = "payments-checkout-order-12345"
task_queue  = "payments-checkout-tq"
```

## Production Safeguards

### Authorizer (Open Source)
> "Use a custom Authorizer on your Frontend Service to set restrictions on who can create, update, or deprecate Namespaces."

Without one, the **`nopAuthority`** authorizer permits all API calls unconditionally. See [[Temporal SH 07 - Security]] and [[Temporal CL 07 - operator]].

### Deletion Protection (Cloud)
Enable deletion protection for production namespaces — prevents accidental removal via UI or API.

### High Availability (Cloud)
For business-critical use cases, **enable High Availability features for a 99.99% contractual SLA**. Pair with [[Temporal BP 08 - Security Controls]].

### Infrastructure as Code (Cloud)
> "Use the Temporal Cloud Terraform provider to manage Namespaces."

Apply `prevent_destroy = true` in Terraform to complement Cloud-level deletion protection:

```hcl
resource "temporalcloud_namespace" "payments_prod" {
  name = "payment-checkout-prd"
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

## Tagging (Temporal Cloud)

Tags complement naming by adding metadata that doesn't fit in the name:

| Tag | Values |
|---|---|
| `environment` | dev, staging, production |
| `team` | platform, payments, identity |
| `division` | engineering, finance, ops |
| `criticality` | high, medium, low |
| `data-sensitivity` | pii, pci, public |
| `latency-sensitivity` | realtime, batch, async |

## SDK Client Configuration

> "Set Namespaces in your SDK Client to isolate your Workflow Executions. If you do not set a Namespace, all Workflow Executions started using the Client will be associated with the `default` Namespace."

**Always register the namespace before configuring it in clients.** Forgetting this is a common bring-up bug.

## ⚠️ Anti-Patterns

⚠️ Mixing case styles (`PaymentsProd` vs `payments-prod`) — breaks when migrating between OSS and Cloud.
⚠️ Letting Workflows fall through to the `default` namespace.
⚠️ One mega-namespace for the whole org (no chargeback, no isolation, shared APS pool).
⚠️ Running OSS without a custom Authorizer in production — `nopAuthority` permits everything.
⚠️ Managing production namespaces by hand instead of via Terraform.
⚠️ Skipping deletion protection on prod namespaces.
⚠️ Cross-namespace communication via ad-hoc HTTP instead of [[Temporal EN 12 - Namespaces]] + Nexus.

## Cross-Refs

- [[Temporal EN 12 - Namespaces]]
- [[Temporal CL 07 - operator]] — namespace CRUD via CLI
- [[Temporal BP 04 - Multi-Tenant Patterns]] — when to escalate to namespace-per-tenant
- [[Temporal BP 06 - Managing APS Limits]] — APS limits are per-namespace
- [[Temporal BP 07 - Cloud Access Control]] — namespace-level roles
- [[Temporal BP 08 - Security Controls]]
- [[Temporal SH 05 - Production Checklist]]
- [[Temporal SH 07 - Security]]

💡 **Takeaway:** Pick `<use-case>-<domain>-<env>` (lowercase, hyphenated), enforce it in Terraform with `prevent_destroy`, tag for chargeback, and never share prod with non-prod.
