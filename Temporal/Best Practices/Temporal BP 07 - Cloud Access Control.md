# BP 07 — Cloud Access Control

🔑 **Default to API keys + service accounts; reach for mTLS only when mutual auth is a hard requirement.**

## Best Practice 1 — Standardize Authentication Per Operation

Teams should **standardize on auth methods** for each operation type:

- **Connecting Temporal clients** to Temporal Cloud (e.g. Worker processes)
- **Automation** — Cloud Ops API, Terraform provider, Temporal CLI ([[Temporal CL 07 - operator]])

### Default Approach
> "It is recommended for teams to use API keys and service accounts for both operations because API keys are easier to manage and rotate for most teams. In addition, you can control account-level and namespace-level roles for service accounts."

API keys win on:
- Easier rotation
- Simpler distribution
- Account- and namespace-level role control on service accounts

### Alternative — When Mutual Auth Is Required
> "Use mTLS certificates to authenticate Temporal clients to Temporal Cloud and use API keys for automation (because Temporal Cloud Operations API and Terraform provider only supports API key for authentication)."

So in practice: even mTLS shops still use API keys for the automation surface.

## Best Practice 2 — Certificate Filters for Multi-Environment Protection

When deploying across **dev and prod with a shared CA**:

> "Give certificates a common name that matches the namespace. This is not a requirement. If you do this when using the same CA for dev and prod environments, then you can leverage Certificate Filters to prevent access to production."

Pattern:

```text
CN = payment-checkout-dev   → cert can only reach dev namespace
CN = payment-checkout-prd   → cert can only reach prod namespace
```

Without filters, a leaked dev cert can authenticate against prod.

## Credential Rotation — Five Steps to Near-Zero Downtime

1. **Generate new credentials** before expiration.
2. **Configure dual credential support** — accept old and new in parallel.
3. **Migrate Worker applications** to the new credential.
4. **Validate connectivity** — confirm every Worker is on new creds.
5. **Remove old credentials** only after successful migration.

Skip step 2 and you create an availability gap during rotation.

## Roles & Permissions

Apply **preconfigured roles** at account and namespace levels — see [[Temporal TC 06 - Account Access]]:

- Account Owner
- Finance Admin
- Global Admin
- Developer
- Read-Only

Assign **namespace-level permissions by job function**, not by team blanket.

## Service Accounts for Non-Human Access

- One service account **per CI/CD pipeline or backend service** — no shared credentials.
- Generate **unique API keys per application or microservice**.
- Restrict each account to the **minimum required permissions**.
- Track which key belongs to which service so you can revoke surgically.

## User Lifecycle

- **SCIM** or the Temporal Cloud API for account lifecycle management.
- Ensure **timely access removal** during role transitions or departures.
- Conduct **regular access reviews**; remove unnecessary accounts.

## ⚠️ Anti-Patterns

⚠️ One shared API key for the whole CI/CD fleet.
⚠️ Same CA for dev + prod with no certificate filters.
⚠️ Hot-swapping credentials with no dual-credential window — mass Worker disconnects.
⚠️ Hard-coding API keys in repos or container images.
⚠️ Granting Global Admin "to make things easier."
⚠️ Onboarding via email invites with no SCIM, then forgetting offboarding.
⚠️ mTLS for everything including automation that doesn't support it.
⚠️ Skipping access reviews — drift accumulates fast.

## Cross-Refs

- [[Temporal TC 06 - Account Access]] — roles surface
- [[Temporal BP 08 - Security Controls]] — encryption, mTLS, key handling
- [[Temporal BP 05 - Managing Namespaces]] — namespace-level permissions
- [[Temporal SH 07 - Security]] — auth on self-hosted
- [[Temporal CL 07 - operator]] — CLI for namespace + access ops
- [[Temporal SH 05 - Production Checklist]]

💡 **Takeaway:** API keys + service accounts by default, mTLS when you need mutual auth, Certificate Filters when sharing a CA across environments, and dual-credential rotation always.
