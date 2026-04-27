# BP 08 — Security Controls

🔑 **Encrypt at the client (codec + failure converter), authenticate via SAML SSO + mTLS/API keys, and isolate via private networking and per-namespace credentials.**

## Identity & Access Management

### SAML SSO
- "Integrate Temporal Cloud with your organization's identity provider via SAML 2.0 for centralized authentication."
- Enforce corporate login policy (MFA, password complexity).
- Disable social logins via support ticket.

### Least-Privilege Access
- Use preconfigured roles: Account Owner, Finance Admin, Global Admin, Developer, Read-Only.
- Assign namespace-level permissions by **job function**.
- Run **regular access reviews**.

### Automated User Provisioning
- Use **SCIM** or the Cloud API for lifecycle management.
- Enforce timely access removal on role change / departure.

### Service Accounts
- Machine identities for CI/CD and backend services — never shared user creds.
- Unique API key per application / microservice.
- Each account scoped to minimum required permissions.

See [[Temporal BP 07 - Cloud Access Control]] and [[Temporal TC 06 - Account Access]].

## Application Auth & API Access

### mTLS
> "Temporal Cloud secures its gRPC endpoint per Namespace via mutual TLS."

- Upload CA certificates to Temporal Cloud.
- Restrict cert distribution to authorized services only.

### Certificate Management
- Track expiration for client and CA certs.
- Automated rotation: **quarterly for clients, annually for CA**.
- Test certs in staging before prod.
- Use Temporal's ability to upload **new CA certs alongside existing ones** for zero-downtime rotation.

### API Key Handling
- Store in a **secrets manager** — never in code.
- Rotate **at least every 90 days**.
- One key per service / person — **no sharing**.
- Monitor usage; revoke on anomalies.
- Admins can disable all user API keys to enforce mTLS-only policy.

## Network Configuration

### Private Connectivity
- Implement **AWS PrivateLink** or **Google Cloud Private Service Connect**.
- Route Workers and applications through private network paths.
- Eliminate public-internet traversal for sensitive connections.

### Namespace Separation
> "Use Temporal Namespaces to isolate workflows for different environments or teams."

- Stricter network controls for production namespaces.
- Separate credentials across environment tiers.
- Data-visibility boundaries aligned with namespace access.

See [[Temporal EN 12 - Namespaces]] and [[Temporal BP 05 - Managing Namespaces]].

## Data Protection & Encryption

### Client-Side Encryption (Payload Codec)
- Use Temporal's **data conversion framework + payload codec interface**.
- Encrypt sensitive data **before** it ever reaches Temporal Cloud.
- Maintain key control, rotation, versioning yourself.
- Deploy custom codec plugins in the SDK.
- Optionally deploy **codec servers** for Web UI / CLI payload inspection.

### Failure Converter
> "Temporal's default behavior copies error messages and call stacks as plain text."

- Configure a **Failure Converter** to encrypt `message` and `stack_trace` fields.
- Otherwise, sensitive context leaks via error reporting.

### Data Retention Policies
- Configure retention 1–90 days based on operational needs.
- Shorter retention = less sensitive data sitting at rest.
- Export extended retention (>90 days) to **GCS or S3**.
- Align with org data-governance requirements.

## Availability & Disaster Recovery

| Option | SLA | RTO | RPO |
|---|---|---|---|
| Single-Region | 99.9% | ≤ 8 hours | ≤ 8 hours |
| Same-Region Replication | 99.99% | ≤ 20 min | ≈ 0 |
| Multi-Region Replication | 99.99% | ≤ 20 min | ≈ 0 |
| Multi-Cloud Replication | 99.99% | ≤ 20 min | ≈ 0 |

### Business-Critical Namespaces
- Conduct **impact analysis** to identify revenue-sensitive Workflows.
- Enable HA for namespaces requiring 99.99% SLA.
- Distribute dependencies across **geographically separated regions**.
- Note: **HA enablement doubles consumption costs** — see [[Temporal BP 09 - Cost Optimization]].

## General Security Posture

- Subscribe to **Temporal Trust Portal** for CVE updates and advisories.
- Keep SDKs **current with security patches**.
- Contact `security@temporal.io` for concerns.

## ⚠️ Anti-Patterns

⚠️ Plain-text payloads through Workflows when handling PII / PCI / PHI.
⚠️ No Failure Converter — exception messages leak sensitive context to error reporting.
⚠️ Sharing API keys across services or pipelines.
⚠️ One CA for dev + prod with no Certificate Filters.
⚠️ Public-internet Worker connections for sensitive workloads.
⚠️ Indefinite retention "just in case" — increases blast radius.
⚠️ Enabling HA on every namespace without an impact analysis (doubles cost).
⚠️ Skipping regular access reviews and SDK upgrades.

## Cross-Refs

- [[Temporal BP 07 - Cloud Access Control]]
- [[Temporal BP 05 - Managing Namespaces]]
- [[Temporal SH 07 - Security]]
- [[Temporal SH 05 - Production Checklist]]
- [[Temporal TC 06 - Account Access]]
- [[Temporal EN 12 - Namespaces]]
- [[Temporal EN 11 - Temporal Service]]
- [[Temporal BP 09 - Cost Optimization]] — HA cost trade-off

💡 **Takeaway:** SAML + least-privilege + per-service credentials + client-side codec + Failure Converter + private networking + retention discipline. Skip any of these and you're trusting Temporal Cloud with cleartext you shouldn't.
