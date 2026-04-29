# 12 — RBAC and ABAC

🔑 Authorization comes after authentication. **RBAC** answers "what role does this user have?" **ABAC** answers "do the user's, resource's, and environment's attributes satisfy this policy?" Most real systems are RBAC + a few ABAC rules in disguise.

## RBAC
Users → Roles → Permissions. Static, easy to audit.

```
alice → [admin]      → [billing.read, billing.write, users.invite]
bob   → [member]     → [billing.read]
carol → [billing-ro] → [billing.read]
```

Pros: trivial to reason about, fits enterprise org charts, IdPs/SSO already model it.
Cons: role explosion (`admin-eu-readonly-billing-q4`), can't express "owners can edit only their own docs."

## ABAC
Permission = `policy(subject_attrs, resource_attrs, action, env)`. Rules express conditions.

```
allow read on document
  when subject.tenant == document.tenant
   and (subject.role == "admin" or subject.id == document.owner_id)
   and env.time.hour between 9 and 18
```

Pros: expressive, no role explosion, models multi-tenant + ownership cleanly.
Cons: harder to audit ("who can do X?" requires evaluating policies, not a join), debug-by-policy.

## Multi-tenant Pattern
Every authenticated request carries `tenant_id` (in the JWT). Every authz check, every DB query, every cache key includes it. ABAC rule: `subject.tenant_id == resource.tenant_id` is the spine; RBAC layers roles on top within a tenant.

⚠️ Putting `tenant_id` only in the application layer is a one-bug-from-disaster setup. Defense in depth: RLS in Postgres ([[Auth 06 - Supabase Auth]]), or a query-builder that always injects the tenant filter.

## Policy Engines
| Engine | Style | Notes |
|---|---|---|
| **OPA / Rego** | ABAC, declarative | CNCF, k8s-friendly, sidecar pattern |
| **Cedar** (AWS) | ABAC + RBAC, typed policies | Powers Verified Permissions |
| **Cerbos** | RBAC + ABAC, YAML rules | Ergonomic, sidecar/embedded |
| **Casbin** | Many models (RBAC, ABAC, ACL) | Library, embedded in app |
| **SpiceDB / OpenFGA** | ReBAC (Zanzibar) | Relationship-based, Google Drive style |

## ReBAC (relationship-based)
Google Zanzibar: model permissions as graph edges. `document:42#owner@user:alice`, `document:42#parent@folder:7`, `folder:7#viewer@group:eng`. Inheritance and delegation fall out naturally — best fit when "share with X" semantics dominate (Notion, Drive, Linear).

## Pragmatic Defaults
Start with RBAC + tenant scoping in JWT claims. Add ownership checks in code (`resource.owner_id == subject.id`). Pull ABAC into a policy engine when you have ≥ 5 scattered checks. Reach for ReBAC only if your domain *is* a sharing graph.

💡 See [[Auth 02 - JWT vs Sessions]] for where these claims live, [[Auth 04 - WorkOS]] FGA for hosted ReBAC.

## Tags
[[Auth]] [[RBAC]] [[ABAC]] [[Authorization]] [[OPA]] [[Multi-tenant]]
