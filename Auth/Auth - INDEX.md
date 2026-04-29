# Auth Knowledge Pack

Refresher vault on identity, authentication, and authorization — protocols, patterns, and the four or five managed services you'll actually pick between. Code-first, terse — for re-reading, not first-time learning.

🔑 **Two questions, one stack:** *Who is this?* (authentication — OAuth/OIDC, SAML, passwords, passkeys) and *What can they do?* (authorization — RBAC/ABAC, RLS, policy engines). Pick a managed provider for the first, design the second yourself around your data model.

## Protocols & Primitives

| # | Note | What it covers |
|---|---|---|
| 01 | [[Auth 01 - OAuth 2.0 and OIDC]] | OAuth = authz, OIDC = authn; access vs id vs refresh tokens; flows |
| 02 | [[Auth 02 - JWT vs Sessions]] | Opaque sessions vs self-contained JWT; revocation, blast radius, hybrid |
| 03 | [[Auth 03 - PKCE]] | Code verifier/challenge, why required for public clients |
| 09 | [[Auth 09 - SAML and SSO]] | SAML 2.0 IdP/SP flows, assertion validation, SCIM provisioning |

## Methods

| # | Note | What it covers |
|---|---|---|
| 10 | [[Auth 10 - Magic Links and Passwordless]] | Email link/OTP, token hygiene, inbox-as-credential threats |
| 11 | [[Auth 11 - MFA and Passkeys]] | TOTP, WebAuthn/passkeys, recovery codes, phishing resistance |

## Authorization

| # | Note | What it covers |
|---|---|---|
| 12 | [[Auth 12 - RBAC and ABAC]] | Roles vs attributes, multi-tenancy, OPA/Cerbos/Cedar/SpiceDB |

## Providers

| # | Note | Best for |
|---|---|---|
| 04 | [[Auth 04 - WorkOS]] | Enterprise SSO + SCIM as a bolt-on (B2B) |
| 05 | [[Auth 05 - Clerk]] | Drop-in React UI, organizations, fast B2C/B2B-SaaS |
| 06 | [[Auth 06 - Supabase Auth]] | Auth tied to Postgres + RLS |
| 07 | [[Auth 07 - Better Auth]] | TS-first, plugin-driven, self-host |
| 08 | [[Auth 08 - Auth0]] | Mature managed identity, Universal Login, Actions |

💡 **Reading order:** start at [[Auth 01 - OAuth 2.0 and OIDC]] / [[Auth 02 - JWT vs Sessions]] for the model, [[Auth 03 - PKCE]] / [[Auth 11 - MFA and Passkeys]] before shipping public-client auth, [[Auth 09 - SAML and SSO]] when the first enterprise customer asks, and [[Auth 12 - RBAC and ABAC]] before the second `is_admin` boolean creeps into your schema. Pick a provider note based on stack: Next.js B2C → [[Auth 05 - Clerk]]; Postgres-heavy → [[Auth 06 - Supabase Auth]]; enterprise-first → [[Auth 04 - WorkOS]]; self-host TS → [[Auth 07 - Better Auth]]; everything else → [[Auth 08 - Auth0]].

## Tags
[[Auth]] [[OAuth]] [[OIDC]] [[JWT]] [[SAML]] [[Passkeys]] [[WebAuthn]] [[RBAC]] [[WorkOS]] [[Clerk]]
