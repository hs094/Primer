# 09 — SAML and SSO

🔑 SAML 2.0 is the XML-based SSO protocol enterprise IT runs on. Old, verbose, signed up the wazoo — but it's what Okta/Azure AD/ADFS speak, so every B2B app eventually implements it.

Source: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf

## Vocabulary
| Term | Meaning |
|---|---|
| **IdP** | Identity Provider — Okta, Azure AD, Google Workspace, ADFS |
| **SP** | Service Provider — your app |
| **Assertion** | Signed XML doc carrying the user's identity + attributes |
| **Metadata** | XML doc describing endpoints, certs; exchanged once at setup |
| **NameID** | Stable user identifier (email, persistent UUID, transient) |
| **ACS URL** | Assertion Consumer Service — SP endpoint that receives the assertion |

## Two Flows
| Flow | Trigger |
|---|---|
| **SP-initiated** | User starts at your app → redirected to IdP → back with assertion |
| **IdP-initiated** | User clicks your app tile in Okta dashboard → IdP posts assertion straight to your ACS |

⚠️ IdP-initiated has no `RelayState` you started — it's vulnerable to assertion replay if you don't validate timestamps + audience strictly. SP-initiated is preferred when you can choose.

## Bindings
- **HTTP-Redirect** — request via URL query string (used for AuthnRequest).
- **HTTP-POST** — assertion posted as form field (used for the response, since assertions are huge).

## What You Must Validate On An Assertion
1. **Signature** — XML-DSig over the `<Assertion>` (or response), against IdP's cert from metadata.
2. **`Issuer`** matches the configured IdP entity ID.
3. **`Audience`** matches *your* SP entity ID.
4. **`NotBefore` / `NotOnOrAfter`** — within clock skew.
5. **`SubjectConfirmation` Recipient** = your ACS URL.
6. **`InResponseTo`** matches the AuthnRequest ID you sent (SP-initiated only).

## SCIM (provisioning, the other half of "enterprise SSO")
SAML logs users in. **SCIM 2.0** (RFC 7644) keeps your user database in sync — IdP pushes `POST /Users`, `PATCH /Users/{id}`, `DELETE` as employees join/leave/move teams. SSO without SCIM means manual offboarding.

💡 In practice, never write SAML by hand — use [[Auth 04 - WorkOS]], a SAML library (`python3-saml`, `passport-saml`), or your IdP/Auth0/Clerk's enterprise tier. XML signature canonicalization bugs are a category of CVE.

## Tags
[[Auth]] [[SAML]] [[SSO]] [[SCIM]] [[Enterprise Auth]] [[Okta]]
