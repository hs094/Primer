# TC 06 — Account Access

🔑 **Two tiers: account-level role (what you can do to the account) + per-namespace permission (what you can do inside a namespace). Three principal types: User, User Group, Service Account.**

## The Two-Tier Model

```
Principal (User | Group | Service Account)
   ├── Account-level role  ──> manage users, billing, namespace creation
   └── Namespace-level permissions  ──> Read / Write / Namespace Admin per NS
```

Account roles govern **the account** (who can invite users, see bills, create namespaces). They do *not* govern day-to-day workflow operations inside a namespace — that's namespace permissions. Detailed matrix in [[Temporal TC 07 - Roles and Permissions]].

## Principals

**User** — a human, identified by email, MFA-protected, optionally federated through SAML.

**User Group** — a named bag of users; namespace permissions assigned to the group cascade to members. Drives consistent "team X has Write on namespace Y" patterns. Often synced from your IdP via SCIM. See [[Temporal TC 08 - Users and Groups]].

**Service Account** — a machine identity. Owns API keys, has its own role + namespace permissions, doesn't take an email. Use for workers, CI, integration scripts — never share a human's API key with a machine.

⚠️ **One email = one account.** A given email address can only belong to one Temporal Cloud account at a time. Need to access two accounts? Use two emails (e.g. via address aliasing).

## Account Owner

The user who creates the account becomes Account Owner — full control across the account, including users, billing, and all namespaces.

⚠️ **Promote a second human Account Owner immediately.** If the sole Owner's email is deactivated (someone leaves), recovery requires Temporal Support. Prefer real human accounts (each with MFA) over a shared mailbox for Owner role. See [[Temporal BP 07 - Cloud Access Control]].

## Inviting Users

UI: Settings → Users → **Invite User**:

1. Email
2. Account role (Owner / Global Admin / Developer / Finance Admin / Read-Only)
3. Per-namespace permissions (Read / Write / Namespace Admin), or none

Invitee accepts via email link, sets up MFA, signs in.

```bash
# tcld equivalent
tcld user invite \
  --email alice@acme.com \
  --account-role developer \
  --namespace-permission "payments-prod.abc12=Write"
```

## Removing Users

```bash
tcld user delete --user-email alice@acme.com
```

Active sessions terminate, API keys owned by the user are invalidated, namespace permissions are removed.

## Service Accounts

Created at account level, optionally scoped to specific namespaces.

```bash
tcld serviceaccount create \
  --name worker-payments-prod \
  --account-role developer \
  --namespace-permission "payments-prod.abc12=Write"

tcld apikey create \
  --service-account-id <sa-id> \
  --name initial-key \
  --duration 90d
```

Permissions for managing SAs:
- **Account Owner / Global Admin** — manage all account-scoped SAs
- **Namespace Admin** — manage SAs scoped to their namespace
- An SA can rotate its own keys (mint new), but cannot delete keys without admin help — preserves audit trail. See [[Temporal TC 04 - API Keys]].

## Identity Federation

**SAML** for SSO — point Cloud at your IdP (Okta, Azure AD, Google Workspace, etc.); users sign in with corporate creds, MFA, conditional access — all enforced by your IdP. Configured at the account level once.

**SCIM** for provisioning — IdP pushes user/group changes to Cloud. Add a user to a group in Okta, they appear in Cloud with the group's permissions. Remove them, they're deprovisioned automatically.

Setup details: [[Temporal TC 09 - SAML and SCIM]].

⚠️ **Don't mix manual and SCIM users without a plan.** If SCIM is the source of truth, drift creeps in fast when ops creates users by hand. Pick one workflow.

## Account Recovery

Cloud supports several recovery paths through the UI / Support:

- **MFA recovery** via recovery codes (saved at MFA setup) or email verification
- **Password reset** through profile settings
- **Email domain change** — requires Temporal Support (e.g. corp domain rename)

Save recovery codes somewhere durable when enabling MFA. Without them and without email access, only Support can rescue an Owner account.

## Where Each Lever Lives

| Action | Where |
|---|---|
| Invite user | UI Settings → Users; `tcld user invite` |
| Set account role | UI / `tcld user set-role` |
| Set namespace permission | UI Namespace edit; `tcld namespace permissions` |
| Create User Group | UI Settings → Groups; SCIM |
| Create Service Account | UI / `tcld serviceaccount create` |
| Configure SAML | UI Settings → SSO |
| Configure SCIM | UI Settings → SCIM |

## Audit Logging

Every account/namespace mutation can be exported via the audit log sink (S3 / GCS, configured at account level). Wire this into your SIEM — see [[Temporal BP 08 - Security Controls]]. Differs from self-hosted, where you'd build this yourself ([[Temporal SH 07 - Security]]).

## Best Practices

- Two+ Account Owners, real humans, MFA on
- Use Groups (not direct user assignments) for namespace permissions wherever possible — fewer audits, easier offboarding
- Service Accounts for *every* non-human caller; never reuse a human's API key for a worker
- Default to least privilege: Read on everything → grant Write only where actually needed
- Federate via SAML + SCIM as soon as you have >5 users
- Review namespace permissions quarterly; audit Service Account scopes monthly

See [[Temporal BP 07 - Cloud Access Control]] for the full posture playbook.

💡 **Takeaway:** Account roles control the account; namespace permissions control workflows. Three principal types (User, Group, Service Account) plus federation (SAML/SCIM) cover every realistic access pattern. Promote a second Owner on day one and lean on Groups for namespace assignments.
