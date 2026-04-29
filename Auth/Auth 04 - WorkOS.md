# 04 — WorkOS

🔑 WorkOS is the "enterprise-ready" layer most apps bolt on once they start selling to companies — SSO, SCIM, audit logs, behind one API.

Source: https://workos.com/docs

## Product Surface
| Product | What It Does |
|---|---|
| **AuthKit** | Hosted login UI + session — full auth platform from first user to SSO |
| **SSO** | One API for SAML and OIDC across every IdP (Okta, Azure AD, Google, Ping, …) |
| **Directory Sync** | SCIM + HRIS — provision/deprovision users and groups automatically |
| **Admin Portal** | Self-serve setup flow you embed; IT admins configure SSO/SCIM themselves |
| **Audit Logs** | Ingest, store, and export structured audit events |
| **FGA / RBAC** | Fine-grained authorization (Zanzibar-style relationships) |

## Why Apps Pick WorkOS
- **Enterprise checklist in days, not quarters.** "Does it support SAML?" → yes. "SCIM?" → yes. "Audit log export?" → yes.
- **One integration, every IdP.** No per-customer SAML metadata wrangling.
- **Admin Portal removes you from the loop.** Customer's IT admin self-configures via your branded URL.
- Free tier covers SSO + Directory Sync up to 1M MAU — pricing kicks in at scale.

## Mental Model
WorkOS sits *between* your app and your customer's IdP:

```
Your App  ⇄  WorkOS API  ⇄  Customer IdP (Okta / Azure AD / …)
```

You speak one OIDC-shaped flow to WorkOS; WorkOS speaks SAML/OIDC/SCIM to whatever the customer runs.

## Compared To Alternatives
- vs **Auth0**: WorkOS is leaner, B2B-first, transparent pricing on enterprise features.
- vs **Clerk**: Clerk is consumer/B2B SaaS UI; WorkOS owns the *enterprise integration* end.
- vs DIY SAML: don't.

⚠️ AuthKit is opinionated — if you need full custom UI/sessions, use the lower-level SSO API and bring your own session layer.

💡 Common pattern: Clerk/Auth0 for B2C login, WorkOS for the "enterprise SSO" tier. See [[Auth 05 - Clerk]], [[Auth 09 - SAML and SSO]].

## Tags
[[Auth]] [[WorkOS]] [[SSO]] [[SAML]] [[SCIM]] [[Enterprise Auth]]
