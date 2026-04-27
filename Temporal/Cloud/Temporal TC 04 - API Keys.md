# TC 04 — API Keys

🔑 **API key → Identity (user or Service Account) → RBAC. Bearer-token auth, easy to rotate, capped at 2-year lifetime.**

## Auth Flow

```
client → API key (Bearer)  →  Cloud authn  →  Identity  →  RBAC check  →  allow/deny
```

The key authenticates *who you are*; the identity (user or Service Account) plus its assigned roles ([[Temporal TC 07 - Roles and Permissions]]) authorizes *what you can do*. Keys never carry permissions directly — they're just identity proofs.

## When to Use API Keys

Use API keys (vs [[Temporal TC 05 - mTLS Certificates]]) when:
- You want simple, fast onboarding
- You don't already operate a private CA
- You need easy rotation without coordinating cert distribution
- You're authenticating CI, scripts, or short-lived workers

Use mTLS instead when you have an existing PKI you trust and want zero-trust transport-level identity.

## Where API Keys Work

- Temporal CLI (`temporal`)
- SDKs: Go, Java, Python, TypeScript, .NET (PHP/Ruby check current docs)
- `tcld` (admin CLI)
- Cloud Ops API (gRPC)
- Terraform provider

## Creating a Key

**Personal key (UI):** Profile → API Keys → **Create API key**. Provide name, description, expiration. The full key string is shown **once** — copy it now.

**Personal key (tcld):**

```bash
tcld apikey create \
  --name worker-prod-2026q1 \
  --description "prod payments worker fleet" \
  --duration 90d
```

**Service Account key:**

```bash
tcld apikey create \
  --name ci-deployer \
  --service-account-id sa-abc123 \
  --duration 365d
```

## Permissions Around Key Management

| Who | Manages |
|---|---|
| Any user | Their own keys |
| Global Admin / Account Owner | Account-scoped Service Account keys |
| Namespace Admin | Namespace-scoped Service Account keys |

Keys inherit the creator's permissions; they cannot escalate. A Developer creating a key for themselves still gets a Developer-scoped key.

## Using a Key

```bash
export TEMPORAL_API_KEY="<key>"
temporal workflow list \
  --address payments-prod.abc12.tmprl.cloud:7233 \
  --namespace payments-prod.abc12
```

```python
# Python
client = await Client.connect(
    "payments-prod.abc12.tmprl.cloud:7233",
    namespace="payments-prod.abc12",
    api_key=os.environ["TEMPORAL_API_KEY"],
    tls=True,
)
```

Go: `client.NewAPIKeyStaticCredentials(apiKey)` in `client.Options.Credentials`. tcld accepts `--api-key` or `TEMPORAL_API_KEY`.

## Limits

- Max **10** non-expired keys per user
- Max **20** non-expired keys per Service Account
- Max expiration **2 years** from creation
- Expired keys can't be revived — create a new one

## Rotation Without Downtime

The canonical four-step rotation:

```bash
# 1. Create the new key
tcld apikey create --name worker-2026q2 --duration 90d
# → NEW_KEY=...

# 2. Roll workers/clients to a config that accepts BOTH old and new
#    (most SDKs let you swap credentials without restart)

# 3. Verify traffic on new key, watch metrics for auth errors

# 4. Delete old key
tcld apikey delete --id <old-key-id>
```

Build your worker config to **read the key on each connection attempt** (env var, secrets-manager fetch) — not bake it into the binary at build time. That makes step 2 a config push, not a redeploy.

## Disable vs Delete

- **Disable** — key remains, can't authenticate. Reversible. Use for "freeze pending investigation."
- **Delete** — gone. Irreversible. Active connections immediately fail.

⚠️ **Deleting a worker's key kills the worker.** Workflow Executions parked on that worker's task queue stall until a new worker (with a valid key) comes up. Always rotate, don't delete-and-recreate.

## Security Practices

- Treat keys like passwords. **Don't commit them** — use Secrets Manager / Vault.
- Rotate every 30–90 days; cap expiration well below the 2-year ceiling.
- One key per workload (worker fleet ≠ CI ≠ laptop).
- Service Accounts for *every* non-human caller — see [[Temporal TC 08 - Users and Groups]].
- See [[Temporal BP 07 - Cloud Access Control]] and [[Temporal BP 08 - Security Controls]] for the full posture.

⚠️ **Key shown once.** On creation, the full secret displays exactly once. Lost it? Create a new one and delete the old.

⚠️ **Service Accounts can self-rotate, not self-delete.** An SA can mint a new key before expiry but cannot delete its own keys — preserves audit trail.

## Namespace-Level API Key Auth + Troubleshooting

A Namespace created with API key auth (vs mTLS) accepts keys at `<ns>.<acc>.tmprl.cloud:7233` for any identity with Namespace permissions — see [[Temporal TC 03 - Cloud Namespaces]].

- **`Unauthenticated`** — key missing, expired, disabled, or whitespace.
- **`PermissionDenied` after auth** — RBAC issue ([[Temporal TC 07 - Roles and Permissions]]).
- **Worker reconnect loop** — check key TTL; courtesy alerts fire near expiry.

💡 **Takeaway:** API keys are the simplest auth path — short-lived bearer tokens tied to a user or Service Account. Build for rotation from day one (config-driven, not baked-in), use Service Accounts for machines, and never let a hot key live longer than 90 days.
