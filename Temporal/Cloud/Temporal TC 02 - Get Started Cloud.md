# TC 02 — Get Started Cloud

🔑 **Sign up → create Namespace → pick auth (API key or mTLS) → connect SDK → invite team. Five steps, ~30 minutes.**

## 1. Sign Up

Three paths into a Cloud account:

- **Direct** — `temporal.io` signup form, credit card or contract.
- **AWS Marketplace** — Pay-As-You-Go, billed via your AWS account.
- **GCP Marketplace** — Contract-only, talk to sales.

The first user to sign up becomes **Account Owner** (full account RBAC, billing, all Namespaces). Use a group alias email (`platform@yourcorp.com`), not a personal mailbox — Account Owner is structural and permanent.

⚠️ **One email per account.** A given email can belong to exactly one Cloud account. If you need to belong to two accounts (consulting, partner orgs), use a different email per account.

## 2. Create a Namespace

UI: Namespaces → **Create Namespace**.

Required:
- **Name** — 2–39 chars, lowercase letters/numbers/hyphens, must start with a letter. Becomes part of `<name>.<account_id>` (e.g. `payments-prod.abc12`).
- **Cloud + Region** — AWS or GCP, pinned for the life of the Namespace.
- **Retention** — 1 to 90 days of Event History. Pick the smallest value your replay/debug story tolerates; it bills.
- **Auth method** — API keys *or* mTLS. Cannot mix on a single Namespace.

Optional:
- CA cert PEM bundle (mTLS path)
- Codec Server URL (decoded payloads in UI)
- Custom Search Attributes

The user who creates a Namespace becomes its Namespace Admin automatically.

See [[Temporal TC 03 - Cloud Namespaces]] for naming conventions and best-practice scoping.

## 3. Configure Authentication

**Path A — API key (easiest, recommended for most teams).** See [[Temporal TC 04 - API Keys]].

```bash
# Profile → API Keys → Create API key (UI)
# Or via tcld:
tcld apikey create \
  --name local-dev \
  --description "laptop dev" \
  --duration 30d
# → prints key ONCE. Copy it now.

export TEMPORAL_API_KEY="<paste>"
```

**Path B — mTLS (existing PKI).** See [[Temporal TC 05 - mTLS Certificates]].

```bash
# Generate CA + leaf if you don't already have one
tcld gen ca   --org acme -d 1y --ca-cert ca.pem --ca-key ca.key
tcld gen leaf --org acme -d 364d \
  --ca-cert ca.pem --ca-key ca.key \
  --cert client.pem --key client.key

# Upload CA bundle to the namespace (UI: Edit → Authentication, paste PEM)
```

## 4. Connect SDK / Worker

Use the **Namespace endpoint** (auto-routes to active region), not the regional endpoint:

```
<namespace>.<account>.tmprl.cloud:7233
```

Python with API key:

```python
from temporalio.client import Client

client = await Client.connect(
    "payments-prod.abc12.tmprl.cloud:7233",
    namespace="payments-prod.abc12",
    api_key=os.environ["TEMPORAL_API_KEY"],
    tls=True,
)
```

Go with mTLS:

```go
cert, _ := tls.LoadX509KeyPair("client.pem", "client.key")
c, _ := client.Dial(client.Options{
    HostPort:  "payments-prod.abc12.tmprl.cloud:7233",
    Namespace: "payments-prod.abc12",
    ConnectionOptions: client.ConnectionOptions{
        TLS: &tls.Config{Certificates: []tls.Certificate{cert}},
    },
})
```

Smoke test with `temporal` CLI:

```bash
temporal workflow list \
  --address payments-prod.abc12.tmprl.cloud:7233 \
  --namespace payments-prod.abc12 \
  --api-key "$TEMPORAL_API_KEY"
```

SDKs supported: Go, Java, Python, TypeScript, .NET, PHP, Ruby. Each has its own connection helper.

## 5. Invite the Team

UI: Settings → Users → **Invite User**. Pick:

- **Account role** — Owner, Global Admin, Developer, Finance Admin, Read-Only (see [[Temporal TC 07 - Roles and Permissions]])
- **Namespace permissions** — Read / Write / Namespace Admin per Namespace

For machine identities (CI, workers in shared infra), create **Service Accounts** instead of human users — same RBAC model, separate API key pool. See [[Temporal TC 06 - Account Access]] and [[Temporal TC 08 - Users and Groups]].

For SSO: configure SAML at account level once, then SCIM auto-provisions users from your IdP — [[Temporal TC 09 - SAML and SCIM]].

⚠️ **At least two Account Owners.** If the sole Owner leaves and the email is deactivated, account recovery requires Temporal Support intervention. Promote a second human (real email, MFA on) immediately after signup. See [[Temporal BP 07 - Cloud Access Control]].

⚠️ **Don't paste API keys into worker config files committed to git.** Use a secrets manager and refresh on rotation — see [[Temporal BP 08 - Security Controls]].

## What's Next

- Wire up [[Temporal TC 10 - Cloud Metrics]] in Prometheus/Grafana
- Set deletion protection on prod Namespaces (`tcld namespace lifecycle set --enable-delete-protection true`)
- Review [[Temporal TC 11 - Billing and Usage]] before scaling traffic
- Plan rotation cadence: 30–90 days for API keys, well before cert expiry for mTLS

💡 **Takeaway:** Spin up a Namespace, pick API key auth (simplest path), connect a worker, invite teammates — production posture (mTLS, SSO, deletion protection, audit log) layers on top once the basics work.
