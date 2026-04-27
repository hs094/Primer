# TC 09 — SAML and SCIM

🔑 **SAML signs users in via your IdP; SCIM keeps Cloud's user list and groups in lockstep with that IdP. Configure SAML first, then layer SCIM.**

## SAML

### Availability
SAML 2.0 SSO is gated to **Business, Enterprise, and Mission Critical** plans. You'll need your **Account ID** — the 5–6 character identifier in your user profile dropdown.

### Supported IdPs
The documented integrations are:

- **Microsoft Entra ID** (formerly Azure AD)
- **Okta**

Other SAML 2.0 providers may work but require manual coordination with Temporal Support.

### Configuration shape
The flow is the same for both supported IdPs:

1. Find your Account ID.
2. Configure a SAML application in your IdP with Temporal-supplied URLs.
3. Open a support ticket with the IdP-side metadata for Temporal to wire up.

Microsoft Entra ID expects:

- **Entity Identifier**: `urn:auth0:prod-tmprl:ACCOUNT_ID-saml`
- **Callback URL**: `https://login.tmprl.cloud/login/callback?connection=ACCOUNT_ID-saml`
- **NameID format**: `emailAddress`
- **Claims**: Email, Name

Okta expects:

- **Audience URI (SP Entity ID)**: `urn:auth0:prod-tmprl:ACCOUNT_ID-saml`
- **Single sign-on URL**: `https://login.tmprl.cloud/login/callback?connection=ACCOUNT_ID-saml`
- **Name ID format**: `EmailAddress`

### Handing off to Temporal
Open a support ticket with:

- The sign-in URL from your IdP application
- The X.509 SAML certificate in PEM format
- One or more email domains to map to this connection

Temporal flips the connection on; from that point users in those domains land in your IdP login flow.

⚠️ Once SAML is enforced for a domain, users who previously signed up with passwords or social login must be re-invited under the SAML connection — accounts are keyed on auth method, not email.

## SCIM

### What it does
SCIM (System for Cross-domain Identity Management) automates **user provisioning and access** between your IdP and Temporal Cloud. New hires appear, leavers vanish, and group membership stays current — without anyone clicking around in the Temporal UI.

### Capabilities
- **User creation and onboarding** — new IdP user → new Cloud user.
- **User deletion and offboarding** — IdP deactivation → Cloud removal.
- **Group membership sync** — group changes propagate.
- **Role mapping** — SCIM groups can be mapped to Temporal Cloud roles and namespace permissions, so adding someone to `temporal-prod-admins` in Okta grants them the matching Cloud access.

### Supported IdPs
Documented integrations:

- Okta
- Microsoft Entra ID (Azure AD)
- Google Workspace
- OneLogin
- CyberArk
- JumpCloud
- PingFederate
- Any SCIM 2.0-compliant provider

### Prerequisites
1. **SAML SSO must already be configured** — SCIM rides on the same connection.
2. Identify an IdP administrator who will own the integration.
3. File a support ticket to enable SCIM (paid feature; subscription upgrade required).

### Okta-specific flow
Temporal Support enables the integration and emails a configuration link to your Okta administrator. Once configured, **change events sync within ~10 minutes**. Bind roles to synchronized groups using `tcld` (see [[Temporal TC 14 - Cloud Operations and tcld]]).

### SCIM groups vs user groups
SCIM groups are **read-only inside Temporal Cloud** — only the IdP can create, rename, or modify members. You can still attach roles and namespace permissions to them on the Temporal side. User Groups (the editable kind) coexist; see [[Temporal TC 08 - Users and Groups]] for that distinction.

⚠️ Disabling SCIM doesn't delete the synced users or groups — it only halts further provisioning. Plan deprovisioning carefully if you're switching IdPs.

## Operational notes
- SAML covers **authentication**; SCIM covers **identity lifecycle**. They're additive, not alternatives.
- For machine identities, SAML/SCIM don't apply — use API keys on Service Accounts (see [[Temporal TC 08 - Users and Groups]]).
- Audit changes through [[Temporal TC 14 - Cloud Operations and tcld]] (audit logs capture user/group operations).
- See [[Temporal SH 07 - Security]] and [[Temporal BP 08 - Security Controls]] for how this slots into a broader security posture.

💡 **Takeaway:** Treat SAML as the front door and SCIM as the building directory — SAML decides who gets in; SCIM keeps the list of who works here. Configure them in that order, automate role mapping through SCIM groups, and never let manual user management drift back in.
