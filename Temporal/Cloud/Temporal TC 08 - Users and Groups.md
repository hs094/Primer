# TC 08 — Users and Groups

🔑 **Identity in Temporal Cloud is two-tier (account + namespace) and three-shape (Users, Groups, Service Accounts) — assign once, scope deliberately.**

## Identity model
Cloud has two identity types:

- **Users** — humans, identified by email or ID. Authenticate with the method they originally signed up with (Google, GitHub, password, SAML).
- **Service Accounts** — machines, identified by name or ID, optional email. Authenticate with API keys.

Both pin into the same permission grid: an **account-level role** plus optional **namespace-level permissions** for specific [[Temporal EN 12 - Namespaces]].

## Users

### Account-level roles
- **Account Owner** — full control, billing, ownership transfer. For security, you cannot remove this role from a user without contacting Temporal Support.
- **Global Admin** — manage users, groups, namespaces, billing-adjacent settings.
- **Developer** — operate namespaces they have access to.
- **Read-Only** — observe.
- **Finance Admin** — billing surfaces only (see [[Temporal TC 11 - Billing and Usage]]).

### Inviting users
Account Owners or Global Admins invite via UI (Settings → Create Users), tcld, or the Cloud Ops API.

```bash
tcld user invite \
  --user-email alice@example.com \
  --account-role Developer \
  --namespace-permission "prod.acct123=Write"
```

The Cloud Ops API equivalent is `POST /cloud/users` (CreateUser). See [[Temporal TC 14 - Cloud Operations and tcld]].

⚠️ The new user must sign in using the **same authentication method they signed up with originally** — switching IdPs mid-stream creates a new user.

### Namespace permissions
Per-namespace grants are independent of the account role:

- **Namespace Admin** — manage namespace settings, retention, certs/keys.
- **Write** — start, signal, terminate, query Workflows.
- **Read** — observe Workflows and history.

Manage from Settings (per-user) or the Namespaces page (per-namespace), or via `tcld user set-namespace-permissions` / the SetUserNamespaceAccess API.

### Removing users
UI Delete, `tcld user delete`, or the DeleteUser API. Account Owners and Global Admins only.

## Groups
Groups let you assign roles once and add members — the cure for per-user permission sprawl.

### Naming and creation
Group names are 3–64 characters, lowercase, with hyphens/underscores allowed. Create via UI (Identities → Create Group), tcld, or the Terraform provider.

### Effective permissions
When a user belongs to multiple groups with overlapping scope, **the most permissive role wins**.

### Group flavors
- **User Groups** — Cloud-native, fully editable in the platform.
- **SCIM Groups** — provisioned and owned by your IdP via [[Temporal TC 09 - SAML and SCIM]]. They cannot be created, deleted, or modified inside Temporal Cloud — only their role bindings can be set here.

Both kinds coexist in the same account. Use SCIM for the org chart, user groups for Temporal-specific cohorts (e.g., on-call rotation).

⚠️ Service Accounts cannot be assigned to user groups — bind their roles directly.

### Management access
Only Account Owners and Global Admins manage groups and their namespace permissions.

## Service Accounts
Use a Service Account for any non-human identity — CI runners, [[Temporal EN 06 - Workers]], the Cloud Ops automation, the [[Temporal CL 01 - CLI Overview]] running unattended.

### Authentication
Service Accounts authenticate with API keys, never passwords. Account Owners and Global Admins create and rotate both the account and its keys.

### Permission scopes
Same two-tier model as users:

- **Account-level**: Read, Write, or Admin across the account.
- **Namespace-level**: optional per-namespace grants.

### Namespace-scoped Service Accounts
A specialized variant — one namespace, fixed Read account role, one namespace permission. **Namespace Admins can create and manage these regardless of their account role**, so a developer with namespace ownership can mint workers without escalating to Global Admin. When the namespace is deleted, the scoped account and its keys go with it.

### Lifecycle
Via UI or tcld:

```bash
tcld service-account create \
  --name prod-worker \
  --account-role Read \
  --namespace-permission "prod.acct123=Write"
tcld apikey create --service-account-id <id> --name worker-key
```

Deleting the Service Account auto-deletes its API keys.

💡 **Takeaway:** Humans get Users, machines get Service Accounts, and Groups exist to keep the matrix from exploding. Match identity type to who is actually authenticating, scope to the smallest namespace set, and lean on SCIM once you have an IdP. See [[Temporal BP 07 - Cloud Access Control]] for the rollout playbook.
