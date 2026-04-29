# 07 — Better Auth

🔑 Better Auth is a TypeScript-first, framework-agnostic auth library you self-host inside your own app — like NextAuth, but pluggable, stricter-typed, and not married to Next.js.

Source: https://better-auth.com/docs

## Shape Of The Library
- One `auth` instance configured at server startup; mount its handler at `/api/auth/*`.
- Client SDK gives you typed `signIn`, `signOut`, `useSession`, etc.
- Database adapter (Drizzle, Prisma, Kysely, Mongo) — owns the `user`, `session`, `account`, `verification` tables.
- Plugins extend the schema and the API surface.

## Core Auth (Built-in)
- Email + password (with verification, reset, breach checks)
- Social sign-on (Google, GitHub, Apple, Discord, …)
- Sessions: opaque server-side tokens in HttpOnly cookies (not JWT by default)
- Account linking
- Rate limiting

## Plugins (The Real Story)
| Plugin | Adds |
|---|---|
| `twoFactor` | TOTP, OTP, backup codes |
| `passkey` | WebAuthn registration + auth |
| `magicLink` | Email link sign-in |
| `emailOTP` | One-time codes via email |
| `organization` | Orgs, members, roles, invitations |
| `multiSession` | Multiple concurrent device sessions |
| `oidc-provider` | Make *your* app an OIDC IdP |
| `admin` | User management endpoints |
| `jwt` | JWT mode (replace cookie sessions) |

Each plugin ships its own client hooks + server endpoints; types flow end-to-end.

## Frameworks
Server: Next.js, Remix, SvelteKit, SolidStart, Nuxt, Astro, Express, Hono, Elysia, TanStack Start, Bun. Client: React, Vue, Svelte, Solid, vanilla.

## When To Pick It
- TypeScript stack, want full source-of-truth in your repo.
- Want plugins instead of paid feature flags.
- Need to self-host (data residency, regulated sector).

## Compared To Alternatives
- vs **NextAuth/Auth.js**: Better Auth is younger, more typed, and not Next-centric.
- vs **Clerk/Auth0**: you own the UI, the DB, and the bill — at the cost of doing it yourself.

⚠️ Default sessions are opaque cookies — not JWT. Add the `jwt` plugin if a downstream service needs to verify locally without hitting your DB. See [[Auth 02 - JWT vs Sessions]].

💡 Closest mental model: "Stripe SDK, but for auth" — clean primitives, you compose.

## Tags
[[Auth]] [[Better Auth]] [[TypeScript]] [[Sessions]] [[Plugins]] [[Self-Hosted]]
