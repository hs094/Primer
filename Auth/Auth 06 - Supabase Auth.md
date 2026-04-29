# 06 — Supabase Auth

🔑 Supabase Auth (open-source GoTrue under the hood) is auth tied directly to your Postgres — every JWT becomes a Postgres role context, and **Row Level Security** policies do the authorization.

Source: https://supabase.com/docs/guides/auth

## The Big Idea
Most stacks: app verifies JWT, app filters DB by `user_id`. Supabase: JWT goes *to* Postgres, RLS filters at the row level.

```
Client (JWT)  →  PostgREST  →  Postgres (RLS sees auth.uid())
```

If your RLS policies are right, the database itself enforces authorization. App code can't accidentally leak rows.

## Authentication Methods
- Email + password
- Magic link / OTP (email + SMS)
- OAuth: Google, GitHub, Apple, Discord, Slack, Azure, … (~20 providers)
- Phone OTP (Twilio/MessageBird/Vonage)
- SAML SSO (per-tenant, paid)
- Anonymous sign-in (upgradeable later)

## RLS Integration
The signed JWT carries `sub` (user id) and any custom claims. Inside Postgres:
```sql
create policy "owner reads own rows"
on documents for select
using (auth.uid() = user_id);
```
`auth.uid()` reads the JWT's `sub` claim from the request context. No app code involved.

## MFA
- TOTP (RFC 6238) and WebAuthn/passkeys.
- AAL1 (password) → AAL2 (after second factor); RLS can require `aal = 'aal2'` for sensitive tables.

## Sessions
- JWT access token (1 hr default) + refresh token.
- `supabase-js` rotates automatically; refresh uses reuse-detection to invalidate stolen tokens.

## When To Pick It
- You're already using Supabase as your DB.
- You like RLS and want auth to feed it directly.
- You want self-host parity (GoTrue is open-source, Apache 2.0).

⚠️ Without RLS configured, anon key + PostgREST exposes your tables. RLS is *the* security boundary; never disable it on a public-facing schema.

💡 See [[Auth 02 - JWT vs Sessions]] for the JWT model, [[Auth 12 - RBAC and ABAC]] for policy-engine alternatives to RLS.

## Tags
[[Auth]] [[Supabase Auth]] [[GoTrue]] [[JWT]] [[RLS]] [[Postgres]]
