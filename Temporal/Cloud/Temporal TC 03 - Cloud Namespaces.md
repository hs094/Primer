# TC 03 — Cloud Namespaces

🔑 **A Namespace is the security + isolation + endpoint boundary. Pick scope (use case × env), pick auth (API key xor mTLS), pick retention. Everything else hangs off it.**

## What a Cloud Namespace Is

Same conceptual unit as [[Temporal EN 12 - Namespaces]] in self-hosted: a logical container for Workflow Executions and Task Queues. In Cloud it carries extra config: a unique gRPC endpoint, retention period, auth method, CA bundle (if mTLS), Codec Server URL, Custom Search Attributes, and per-namespace RBAC.

## Identifiers

```
Name:        payments-prod
Account ID:  abc12          (≥5 chars, assigned by Temporal)
Namespace ID: payments-prod.abc12   ← what you pass as --namespace
Endpoint:    payments-prod.abc12.tmprl.cloud:7233
```

**Name rules:** 2–39 chars, lowercase, letters/numbers/hyphens, must start with a letter, end letter or number.

## Endpoints

Two flavors:

```
# Namespace endpoint (recommended) — auto-routes to active region
<namespace>.<account>.tmprl.cloud:7233

# Regional endpoint — shared across namespaces in a region
<region>.<cloud_provider>.api.temporal.io:7233
```

Use the Namespace endpoint unless you specifically need the regional one (typically only for HA failover scenarios — [[Temporal TC 13 - High Availability]]).

⚠️ **mTLS + regional endpoint requires SNI override.** When connecting to the regional endpoint with mTLS, set `server_name` / `tls-server-name` to the *namespace endpoint* hostname so SNI matches the cert. Easier to just use the namespace endpoint.

## Creating a Namespace

UI: Namespaces → **Create Namespace**.

Required:
- Name + cloud + region
- Retention period (1–90 days)
- Auth: API keys *or* mTLS CA bundle

Optional:
- Codec Server endpoint (HTTPS, optional access-token + CORS toggles)
- Custom Search Attributes
- User permissions

The creator gets Namespace Admin automatically.

```bash
# tcld equivalent
tcld namespace create \
  --namespace payments-prod \
  --region aws-us-east-1 \
  --retention-days 14 \
  --ca-certificate-file ./ca.pem
```

## Naming Conventions

Three patterns from the docs, pick one and stay consistent:

```
<use-case>_<environment>                      # billing_prod
<use-case>_<service>_<environment>            # billing_invoicing_prod
<use-case>_<domain>_<environment>             # billing_emea_prod
```

(Note: hyphen vs underscore — Cloud names allow only hyphens, so adapt: `billing-emea-prod`.)

## Scoping Strategy

- **One namespace per use case + env.** A namespace handles many workflow types and high concurrency — don't shard by workflow type.
- **Always isolate prod from non-prod.** Different namespace, different RBAC, different CA if mTLS.
- **Cross-namespace? Use [[Temporal EN 11 - Temporal Service]] Nexus**, not direct calls — clean service contract, no ambient coupling.
- **Mission-critical isolation** — split high-impact workloads into their own namespace to shrink blast radius.

## Editing a Namespace

UI: Namespaces → select → **Edit**:

- Custom Search Attributes (additive; deletes wipe data)
- CA certificates + certificate filters ([[Temporal TC 05 - mTLS Certificates]])
- Codec Server endpoint
- Namespace permissions / user access
- Retention period
- Tags

```bash
# tcld
tcld namespace list
tcld namespace get --namespace payments-prod.abc12
tcld namespace accepted-client-ca set    --namespace payments-prod.abc12 --ca-certificate-file ./ca.pem
tcld namespace certificate-filters import --namespace payments-prod.abc12 --input-file filters.json
```

## Tags

Up to **10 tags per namespace** for org/cost/team labelling.

- Keys + values: 1–63 chars, lowercase letters/numbers/`._-@`
- Unique keys per namespace
- Only **Account Admins / Owners** can create or edit tags
- All namespace users can view

## Deletion

```
UI: Edit → Danger Zone → type "DELETE" to confirm
```

⚠️ **Deletion is immediate and total.** Workflow Executions and Task Queues are wiped on delete. There is no undo, no recycle bin, no recovery window.

**Mitigation — deletion protection:**

```bash
tcld namespace lifecycle set \
  --namespace payments-prod.abc12 \
  --enable-delete-protection true
```

Set this on every prod namespace at create time.

## Limits Worth Knowing

- Retention: 1–90 days max (events past this are archived, not online)
- Up to 16 CA certs per bundle, 32 KB pre-base64 ([[Temporal TC 05 - mTLS Certificates]])
- Up to 25 cert filters per namespace
- Up to 10 tags per namespace
- Custom Search Attributes have per-account caps (check current docs)

## Codec Server + Tooling

Optional HTTPS endpoint Cloud calls when rendering Event History so the UI shows **decoded** payloads. Required if you encrypt end-to-end. Knobs: endpoint URL, pass user access token, include cross-origin credentials.

⚠️ **Codec server runs in your infra.** Cloud calls *out* from the UI session — make it reachable from the browser, not only the worker network.

⚠️ **Auth is immutable per namespace.** API key vs mTLS is fixed at create time. To switch, create a new namespace and migrate.

Tooling: Web UI for ad-hoc, `tcld namespace ...` for CI, Terraform `temporalcloud_namespace` for IaC, Cloud Ops API underneath.

💡 **Takeaway:** Namespace = name + region + retention + auth + RBAC scope. Pick the boundary at use-case × env granularity, turn on deletion protection in prod, and treat the choice between API key and mTLS as load-bearing — you can't change it after creation.
