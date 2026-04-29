# 05 — Clerk

🔑 Clerk is a drop-in auth platform: prebuilt React components, organizations, MFA, social SSO, and a session model — you import a `<SignIn />` and you're done.

Source: https://clerk.com/docs

## What You Get In 5 Minutes
- `<SignIn />`, `<SignUp />`, `<UserProfile />`, `<OrganizationSwitcher />` — production UI components.
- Hosted sign-in page if you don't want to embed.
- Session cookies + JWT verification for your backend.
- Webhooks for user/org lifecycle events.

## Authentication Methods
| Category | Methods |
|---|---|
| Passwords | Email/username + password, with breach checks |
| Social | Google, GitHub, Apple, Microsoft, … (40+) |
| Passwordless | Email link, email code, SMS code |
| MFA | TOTP, SMS, backup codes |
| Passkeys | WebAuthn / FIDO2 |
| Enterprise | SAML SSO (paid tier) |

## Organizations
First-class B2B primitive: a `User` belongs to many `Organizations` with roles + custom permissions. `<OrganizationSwitcher />` handles tenant selection; the active org is encoded in the session JWT (`org_id`, `org_role`, `org_permissions`).

## JWT Templates
Define a custom claim set per backend. Clerk signs and you verify with their JWKS:
```
GET https://<your-instance>.clerk.accounts.dev/.well-known/jwks.json
```
Useful for handing your existing API a JWT with exactly the claims it expects (Hasura, Supabase, custom RBAC).

## Frameworks
First-class SDKs: Next.js (App + Pages), React, Remix, Expo, Vue, Nuxt, Astro, Express, Go, Ruby on Rails. Backend SDKs verify session tokens locally.

## Compared To Alternatives
- vs **Auth0**: Clerk is more opinionated, ships UI, friendlier defaults; less flexible for unusual flows.
- vs **WorkOS**: Clerk leans B2C/B2B-SaaS; WorkOS owns deeper enterprise (full SCIM, audit log export).
- vs **Supabase Auth**: Clerk is auth-only; Supabase ties auth to its DB.

💡 Sweet spot: a Next.js SaaS with users + orgs that wants Stripe-style polish without building it. See [[Auth 04 - WorkOS]] for enterprise-add-on, [[Auth 06 - Supabase Auth]] for the DB-coupled alternative.

## Tags
[[Auth]] [[Clerk]] [[Organizations]] [[Passkeys]] [[JWT]] [[B2B SaaS]]
