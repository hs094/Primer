# 12 — Row Level Security

🔑 RLS pushes per-row authorization into the database — policies are SQL predicates the planner ANDs onto every query.

Source: https://www.postgresql.org/docs/current/ddl-rowsecurity.html

## Enable & Force
```sql
ALTER TABLE docs ENABLE ROW LEVEL SECURITY;
ALTER TABLE docs FORCE  ROW LEVEL SECURITY;  -- also applies to table owner
```
- Without `FORCE`, the table owner bypasses policies.

## Policies
```sql
CREATE POLICY tenant_isolation ON docs
  FOR ALL
  TO app_user
  USING       (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK  (tenant_id = current_setting('app.tenant_id')::uuid);
```
| Clause | Applies to |
|---|---|
| `USING` | Rows the policy can **see/return** (`SELECT`, `UPDATE`, `DELETE`) |
| `WITH CHECK` | Rows the policy lets you **write** (`INSERT`, `UPDATE`) |

💡 If `WITH CHECK` is omitted, it defaults to `USING` — you can read what you can write.

## Multi-Tenant Pattern
```sql
-- Per-request: app sets tenant id once
SELECT set_config('app.tenant_id', '7e2a...', true);  -- true = local to tx
```
- Single Postgres role for app, tenant identity carried in a session GUC.
- Policy reads the GUC; planner often inlines it as a constant.

## Roles & Bypass
- `BYPASSRLS` attribute — superuser-style escape hatch (admin tooling, migrations).
- ⚠️ `SECURITY DEFINER` functions bypass RLS by default — use cautiously.

## Permissive vs Restrictive
- Default = `PERMISSIVE` — multiple policies OR'd.
- `RESTRICTIVE` policies AND on top — used for safety nets ("never see deleted rows").

```sql
CREATE POLICY hide_deleted ON docs AS RESTRICTIVE
  FOR SELECT USING (deleted_at IS NULL);
```

## Performance Notes
- ⚠️ RLS predicates can defeat index choice if not sargable.
- 💡 Index `(tenant_id, ...)` so tenant filter is the leading column.
- Confirm via `EXPLAIN` — RLS predicate appears in the plan.

## Tags
[[PostgreSQL]] [[RLS]] [[Multi-tenant]] [[Security]]
