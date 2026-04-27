# TC 07 — Roles and Permissions

🔑 **Five account roles × three namespace permissions. Account roles never grant day-to-day workflow access — that's namespace permissions, granted separately.**

## The Two Axes

```
Principal
   ├── Account role         — Owner | Global Admin | Developer | Finance Admin | Read-Only
   └── Namespace perm  (per NS)  — Namespace Admin | Write | Read
```

Crucial mental model: **account-level roles do not govern operations inside a namespace.** A Developer with no namespace permission cannot list workflows in any namespace. Account Owners and Global Admins are the exception — they get implicit Namespace Admin everywhere.

## Account-Level Roles

### Account Owner
- Owns and governs the account
- Can create namespaces (auto Namespace Admin on every NS, including ones they didn't create)
- Full **billing, payments, and usage** access
- Can manage all users, groups, service accounts
- Manages audit log sinks and account-wide config
- **Best practice: assign to ≥2 humans** with real direct emails — see [[Temporal BP 07 - Cloud Access Control]]

### Global Admin
- Administers account configuration and users
- Can create namespaces (auto Namespace Admin everywhere, like Owner)
- **Usage-only** billing access (sees consumption, can't change payment)
- Cannot have permissions revoked by anyone except Account Owner
- Can manage connectivity rules, Nexus endpoints
- Workhorse role for platform/SRE leads

### Developer
- Can create namespaces; auto Namespace Admin on **only** the ones they create (revocable)
- No automatic access to other namespaces — must be granted explicitly
- No billing access
- Manages their own API keys (and SA keys per their NS scope)
- Default for application engineers

### Finance Admin
- **Cannot** create namespaces
- Manages billing + payment information exclusively
- No namespace access whatsoever
- For finance/procurement seats

### Read-Only
- Views account config and resources
- Cannot create namespaces, cannot touch billing
- Restricted observation only
- For auditors, observers, junior engineers in evaluation

## Namespace-Level Permissions

Granted **per namespace**, on top of an account role. Three tiers:

| Permission | Workflow ops | Schedules | Search Attributes | Settings | Access control |
|---|---|---|---|---|---|
| **Read** | List, describe, fetch history | View | View | View | — |
| **Write** | Start, signal, cancel, terminate, reset | Create, edit | View | View | — |
| **Namespace Admin** | All Write + batch ops, retention edits | Full | Add/remove | Full | Add/remove users on this NS |

### Read (≈40 ops)
Listing executions, describing workflows, fetching event histories, reading schedules and task queue stats. Pure observation.

### Write (≈70 ops, includes Read)
Adds: start/signal/cancel/terminate workflows, create/edit/delete schedules, poll task queues, manage worker deployments, query workflows. Cannot delete/modify the namespace itself.

### Namespace Admin (≈95 ops, includes Write)
Adds: edit retention, add/remove Custom Search Attributes, manage CA bundle and cert filters, configure codec server, batch operations (terminate/reset many), manage namespace-scoped service accounts and their permissions, delete the namespace (subject to deletion protection).

## Implicit Permissions

| Role | Auto-granted on namespaces |
|---|---|
| Account Owner | Namespace Admin on **every** NS in the account |
| Global Admin | Namespace Admin on **every** NS in the account |
| Developer | Namespace Admin on NSes **they create** (revocable) |
| Finance Admin | None |
| Read-Only | None |

So Developer + Finance Admin + Read-Only roles can *receive* namespace-level permissions, while Owner and Global Admin already have them implicitly.

## Service Accounts

Service Accounts get the same role + permission model as users. Common shapes:

- **Worker SA** — Developer account role + Write on the one namespace it serves
- **CI/deploy SA** — Developer + Namespace Admin (so it can manage Custom Search Attributes during deploys)
- **Observability SA** — Read-Only + Read on all namespaces (for metrics scrapers, dashboards)

Permissions to *manage* SAs:
- Account-scoped SA: Account Owner / Global Admin
- Namespace-scoped SA: any user with **Namespace Admin** on the target NS

## API Key Authorization

> All roles can create and manage their **own** API keys.

Keys inherit the creator's permissions and **cannot escalate** — a Developer's key is forever a Developer-scoped key. To get more privilege, the user has to be granted more privilege first; the key follows.

See [[Temporal TC 04 - API Keys]] for the key lifecycle and [[Temporal TC 06 - Account Access]] for principal management.

## Permission Matrix at a Glance

| Capability | Owner | Global Admin | Developer | Finance | Read-Only |
|---|---|---|---|---|---|
| Manage users / groups | ✅ | ✅ | — | — | — |
| Create namespaces | ✅ | ✅ | ✅ | — | — |
| Implicit Namespace Admin everywhere | ✅ | ✅ | — | — | — |
| Create account-scoped Service Accounts | ✅ | ✅ | — | — | — |
| Audit log sink config | ✅ | — | — | — | — |
| Billing — manage payment | ✅ | — | — | ✅ | — |
| Billing — view usage | ✅ | ✅ | — | ✅ | — |
| Connectivity rules / Nexus endpoints | ✅ | ✅ | — | — | — |
| View account audit log | ✅ | ✅ | — | — | ✅ |

For workflow-level granularity (the 95+ data-plane ops mapped to Read/Write/NSAdmin), see Temporal Cloud's **permissions reference** in their docs.

## Least-Privilege Patterns

- New engineer onboarding → Developer + Read on prod, Write on dev/staging
- Auditor / SRE shadow → Read-Only + Read on every namespace
- Worker fleet → Service Account, Developer role, Write on the one namespace they serve
- CI deployer → Service Account, Developer role, Namespace Admin on the namespaces it deploys to
- Finance/procurement → Finance Admin, no namespace permissions

⚠️ **Don't use Account Owner for daily ops.** Reserve it for rare admin tasks. Day-to-day platform work belongs to Global Admin (no billing footgun).

⚠️ **Removing a user doesn't always retract everything.** Verify their Service Account memberships and any Group memberships are also cleaned up — see [[Temporal TC 08 - Users and Groups]].

⚠️ **Namespace Admin on prod is dangerous.** It includes namespace deletion. Pair with deletion protection (`tcld namespace lifecycle set --enable-delete-protection true`) and require >1 reviewer for destructive admin actions.

## Cross-References

- Day-to-day RBAC ops → [[Temporal TC 06 - Account Access]]
- Group mechanics → [[Temporal TC 08 - Users and Groups]]
- Federated identity → [[Temporal TC 09 - SAML and SCIM]]
- Cert/key auth (separate from authorization) → [[Temporal TC 04 - API Keys]], [[Temporal TC 05 - mTLS Certificates]]
- Posture guidance → [[Temporal BP 07 - Cloud Access Control]], [[Temporal BP 08 - Security Controls]]
- Self-hosted contrast → [[Temporal SH 07 - Security]]

💡 **Takeaway:** Two axes — account role (account ops) and namespace permission (workflow ops) — applied to Users, Groups, and Service Accounts. Default to Developer + explicit per-namespace Write; reserve Account Owner for true administrators; lean on Groups for offboarding hygiene.
