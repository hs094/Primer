# 08 — Auth0

🔑 Auth0 (Okta-owned) is the incumbent managed identity platform — Universal Login, deep customization via Actions, every protocol supported, every SDK shipped.

Source: https://auth0.com/docs

## Anatomy
| Concept | Meaning |
|---|---|
| **Tenant** | Your isolated Auth0 environment (`yourapp.us.auth0.com`) |
| **Application** | A client (SPA, native, M2M, web) — gets `client_id`/`client_secret` |
| **API** | Your resource server, defines `audience` and scopes/permissions |
| **Connection** | An identity source: DB, social, enterprise (SAML/OIDC/AD) |
| **Universal Login** | Hosted, customizable login page (recommended) |
| **Actions** | Node.js code that runs at flow hooks (login, M2M, post-user-reg, …) |

## Flows Supported
Every OAuth/OIDC flow: Auth Code + PKCE, Client Credentials, Device Code, CIBA, Resource Owner (legacy). Plus legacy Rules/Hooks (deprecated for Actions).

## Customization Layers
1. **Branding** — logos, colors, custom domain.
2. **Universal Login templates** — Liquid + custom CSS, or full custom HTML.
3. **Actions** — inject custom claims, deny logins, call external APIs, enforce MFA conditionally.
4. **Forms / pre-built screens** — progressive profiling, consent, etc.

## B2B vs B2C
- **B2C**: a single tenant, social + DB connections, MFA, anomaly detection, breached-password protection.
- **B2B (Organizations)**: each customer-org has its own connections (their SAML/OIDC), branding, members, roles. Token includes `org_id`.

## Strengths
- Maturity — protocol coverage is the deepest in the market.
- Extensibility — Actions cover anything you'd want to script.
- Compliance — SOC2, HIPAA, ISO 27001, FedRAMP options.

## Weaknesses
- **Pricing** — historically painful past the free tier; per-MAU plus fees for enterprise features.
- Complexity surface is large; small teams over-configure.
- Acquired by Okta — overlap with Okta CIC creates roadmap noise.

⚠️ Default access tokens to your *own* API are JWT only if you specify an `audience` matching a registered API. Tokens to the Auth0 Management API are opaque. Easy to confuse.

💡 See [[Auth 04 - WorkOS]] (leaner B2B SSO), [[Auth 05 - Clerk]] (faster B2C UI), [[Auth 01 - OAuth 2.0 and OIDC]].

## Tags
[[Auth]] [[Auth0]] [[OIDC]] [[OAuth]] [[Universal Login]] [[Actions]]
