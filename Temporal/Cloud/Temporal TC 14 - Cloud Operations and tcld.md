# TC 14 — Cloud Operations and tcld

🔑 **Everything Cloud's web UI does is also a public API — drive it with `tcld` for one-off ops, the Terraform provider for declarative state, the Cloud Ops API for code, and lean on audit logs and notifications to know what changed.**

## Cloud Ops API

### What it is
The Cloud Ops API is an **open-source public HTTP and gRPC API for managing Temporal Cloud control-plane resources** — namespaces, users, service accounts, API keys, connectivity rules, and Nexus endpoints. The web UI, `tcld`, and the Terraform provider all sit on top of it.

### Endpoint and auth
- gRPC and HTTP at `saas-api.tmprl.cloud:443`.
- Authenticate with an **API key** tied to a Cloud user or Service Account.
- Send the `temporal-cloud-api-version` header so the server pins the schema.

Many operations require Admin privileges (Account Owner or Global Admin); read-style calls accept lesser roles.

### SDKs and protos
- **Go**: official SDK at `github.com/temporalio/cloud-sdk-go` — protobufs pre-compiled, idiomatic types.
- **Other languages**: pull `.proto` files from the Cloud Ops API repo and run `protoc` for your stack.

### Common automation patterns
- Provision per-environment namespaces from CI.
- Bootstrap user/role/group mappings on day-one.
- Rotate Service Account API keys on a schedule.
- Audit access by enumerating users and their effective namespace permissions.

### Rate limits
- **160 RPS per account** total.
- **40 RPS per user**, **80 RPS per service account**.
- **10 concurrent async operations** per account.

Need more? File a support ticket with use-case detail.

## tcld

### What it is
`tcld` is the **Cloud-side CLI** — distinct from the regular `temporal` CLI in [[Temporal CL 01 - CLI Overview]] which targets the data plane (Workflows, Tasks, Schedules). `tcld` manages the account, namespaces, identities, certificates, and feature flags.

### Install
```bash
# Homebrew (recommended)
brew install temporalio/brew/tcld

# Or build from source (Go 1.18+)
git clone https://github.com/temporalio/tcld
cd tcld && make
sudo cp tcld /usr/local/bin/
tcld version
```

### Auth
```bash
tcld login                # interactive browser flow
# or
export TEMPORAL_CLOUD_API_KEY=...   # API-key auth for scripts
```

### Command groups
- `account` — plan, settings.
- `apikey` — create, list, delete keys.
- `connectivity-rule` — public / PrivateLink / PSC rules (see [[Temporal TC 12 - Connectivity]]).
- `feature` — toggle preview features.
- `generate-certificates` — issue mTLS material.
- `login` / `logout` — auth.
- `namespace` — create, update, set retention, manage CA bundles, set search attributes.
- `nexus` — Nexus endpoints.
- `request` — inspect long-running async requests.
- `user` — invite, set roles, set namespace permissions.
- `version`.

The `--auto_confirm` flag (or `AUTO_CONFIRM=true`) skips prompts for unattended scripts.

```bash
tcld namespace create \
  --namespace billing-prod \
  --region aws-us-east-1 \
  --retention-days 14 \
  --ca-certificate-file ca.pem
```

## Terraform provider

### Configuration
```hcl
terraform {
  required_providers {
    temporalcloud = {
      source = "temporalio/temporalcloud"
    }
  }
}

provider "temporalcloud" {
  api_key = var.temporal_cloud_api_key
}
```

Or `export TEMPORAL_CLOUD_API_KEY=...` and skip the block.

### Supported resources
- `temporalcloud_namespace` — full CRUD plus import.
- `temporalcloud_nexus_endpoint`.
- `temporalcloud_user` — account role + namespace permissions.
- `temporalcloud_service_account` — name-based machine identity.
- `temporalcloud_apikey` — bearer tokens.

### Example
```hcl
resource "temporalcloud_namespace" "billing" {
  name               = "billing-prod"
  regions            = ["aws-us-east-1"]
  accepted_client_ca = base64encode(file("ca.pem"))
  retention_days     = 14
}

resource "temporalcloud_service_account" "worker" {
  name         = "billing-worker"
  account_role = "Read"
  namespace_accesses = [{
    namespace_id = temporalcloud_namespace.billing.id
    permission   = "Write"
  }]
}

resource "temporalcloud_apikey" "worker" {
  service_account_id = temporalcloud_service_account.worker.id
  name               = "billing-worker"
  duration           = "8760h"
}

output "apikey_token" {
  value     = temporalcloud_apikey.worker.token
  sensitive = true
}
# terraform output -json apikey_token
```

### Limits
- Once a resource is Terraform-managed, **only** Terraform should change it; out-of-band edits cause drift.
- **Account Owners** cannot be created or modified via Terraform.
- Imported API keys can only have name/description edited — tokens are unrecoverable post-creation.

## Audit logs

### Scope
**Forensic access information for control-plane operations.** Captures account, API-key, connectivity-rule, namespace, namespace-export, Nexus, service-account, user, and user-group events. Does **not** capture data-plane activity (Workflow Executions live in [[Temporal EN 03 - Workflows]] history and [[Temporal EN 10 - Visibility]]).

Only Account Owners and Global Admins can view, configure, or manage audit logs.

### Schema
JSON, with fields:

- `operation`, `principal` (user, service account, or API key), `status` (`OK` / `ERROR`).
- `emit_time`, `x_forwarded_for`, `log_id`, `version`.
- `raw_details` — request specifics.

Deprecated: `user_email`, `level`, `caller_ip_address`, `details`, `category`.

### Sinks and retention
- **Sinks**: AWS Kinesis, GCP Pub/Sub. Events flow within ~1 hour of setup.
- **In-UI retention**: up to **30 days**.
- **API queries**: time-range filtered, paginated, max 1000 results per request.
- UI: Settings → Audit Logs; download up to 1,000 events locally.

⚠️ For long-term retention, ship to a sink — 30 days in-product isn't a substitute for a SIEM.

## Notifications

### What you get
- **Cloud status updates** — subscribe at the [Cloud status page](https://status.temporal.io/) for incident notifications via email, Slack, SMS, etc.
- **Administrative emails** from `noreply@temporal.io`. Allowlist that sender.

### Event categories
- **Security/Infrastructure** — certificate expiry at 15, 10, 5 days; API-key expiry at 30, 20, 10 days. Sent to Global Admins, Namespace Admins, Account Owners, plus the API-key owner.
- **Billing/Credits** — sign-up credit expiry at 30, 14, 7, 1 day; consumption alerts at 50% and 90%. Sent to Account Owners and Finance Admins (see [[Temporal TC 11 - Billing and Usage]]).
- **Account changes** — plan-type changes, namespace failover succeeded/failed (see [[Temporal TC 13 - High Availability]]).

To change recipients or unsubscribe, file a support ticket — there's no self-serve preferences page yet.

## Putting it together
- Use **Terraform** for the steady-state shape of namespaces, identities, and Nexus endpoints — it survives team turnover.
- Use **`tcld`** for human-driven one-offs, debugging, and bootstrap.
- Use the **Cloud Ops API** when Terraform's resource model isn't enough or you need event-driven automation (e.g., new project → namespace + service account + API key).
- Stream **audit logs** to your SIEM and monitor **notifications** so cert/key expiry never surprises you.
- Cross-check role assignments against [[Temporal TC 08 - Users and Groups]] and [[Temporal BP 07 - Cloud Access Control]].

💡 **Takeaway:** All four control-plane surfaces — Cloud Ops API, `tcld`, Terraform, audit logs — are views of the same state. Pick the right tool per task (declarative for shape, CLI for one-offs, API for automation), and treat audit logs plus notifications as the feedback loop that catches drift and expiring credentials before they bite.
