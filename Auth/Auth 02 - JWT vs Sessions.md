# 02 — JWT vs Sessions

🔑 Two ways to remember a logged-in user: an **opaque session id** (server is the source of truth) or a **self-contained JWT** (token is the source of truth, until it expires).

## Side-by-Side
| Aspect | Opaque Session | JWT |
|---|---|---|
| Storage | Random id in cookie; state in DB/Redis | All claims in the token; signed |
| Lookup | Server hits store on every request | Verify signature + claims, no I/O |
| Revoke | Delete the row — instant | Hard: blocklist `jti`, or wait for `exp` |
| Size | ~32 bytes | 500 B – 2 KB; bloats every request |
| Mutate user | Change the row, next request sees it | Stale until refresh — claims are frozen |
| Multi-service | Each service calls auth | Each service verifies locally with JWKS |

## When Sessions Win
- Web apps with one backend
- Need instant logout, ban, role change
- Sensitive ops (banking, admin)
- You already run Redis/Postgres — the "lookup cost" is a myth at small scale

## When JWTs Win
- Stateless APIs, many microservices, no shared session store
- Mobile / SPA hitting backends across domains
- Federated auth — third party (Auth0/Clerk/WorkOS) issues, you only verify

## The Hybrid (most production systems)
- **Short-lived access JWT** (5–15 min) — stateless, fast verify
- **Refresh token** stored server-side, rotated on use, revocable
- Logout = revoke refresh token; access token dies on its own in minutes

⚠️ "JWT for sessions" — putting a 24-hour JWT in a cookie and calling it done — gives you the size cost of JWT *and* the revocation problem. Almost always wrong.

💡 Default: opaque sessions for monoliths, short-TTL JWTs + refresh for distributed systems. See [[Auth 04 - WorkOS]], [[Auth 05 - Clerk]] for managed versions.

## Tags
[[Auth]] [[JWT]] [[Sessions]] [[Cookies]] [[Refresh Token]]
