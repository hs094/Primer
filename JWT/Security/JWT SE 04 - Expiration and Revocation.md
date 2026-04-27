# SE 04 — Expiration and Revocation

🔑 **One-line: short `exp`, validated `nbf`/`iat`, and a `jti`-keyed blocklist — JWTs aren't sessions, they're bearer credentials with a deadline.**

## Threat

JWTs are stateless: the server doesn't track them, so a leaked token is valid until it expires. Without proper time-claim handling, "valid until expires" becomes "valid forever in practice."

The relevant claims from RFC 7519 §4.1 ([[JWT FN 03 - Registered Claims]]):

> §4.1.4 (`exp`): "The 'exp' (expiration time) claim identifies the expiration time on or after which the JWT MUST NOT be accepted for processing."

> §4.1.5 (`nbf`): "The 'nbf' (not before) claim identifies the time before which the JWT MUST NOT be accepted for processing."

> §4.1.6 (`iat`): "The 'iat' (issued at) claim identifies the time at which the JWT was issued. This claim can be used to determine the age of the JWT."

> §4.1.7 (`jti`): "The 'jti' claim provides a unique identifier for the JWT... The claim can be used to prevent the JWT from being replayed."

Concrete failure modes:

- **No `exp`** — token works for years; one phishing attempt = permanent compromise.
- **`exp` checked but not enforced** — library returns parsed claims even when expired (e.g. `verify_exp=False`).
- **24h+ access tokens** — an attacker who steals the token from a browser cache or a log line owns the account for the rest of the day.
- **No revocation path** — user clicks "Sign out everywhere" and nothing happens until tokens naturally expire.
- **No `jti` / no replay protection** — the same token replays indefinitely against high-value endpoints.
- **Missing `nbf` validation on pre-issued tokens** — a token timestamped for tomorrow is accepted today.

```http
GET /api/wallet/withdraw HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyXzQyIiwiZXhwIjoxNzY0MDAwMDAwLCJpYXQiOjE3NjMwMDAwMDB9.sig
```

If `exp` is six months in the future and the verifier never checks it, that bearer token is a permanent withdraw key.

RFC 8725 §2 frames the broader risk: any property of a JWT that is not validated becomes an attack surface. Time claims are the simplest example.

## Mitigation

Validate every time claim, keep TTLs short, and add a revocation lane for the cases the deadline can't cover.

```python
import jwt  # [[JWT PR 01 - PyJWT]]

claims = jwt.decode(
    token,
    key=public_key,
    algorithms=["RS256"],
    audience="https://api.example.com",
    issuer="https://auth.example.com",
    leeway=30,                         # RFC 7519 §4.1.4 — small clock-skew window
    options={
        "require": ["exp", "iat", "iss", "aud", "jti"],
        "verify_exp": True,
        "verify_iat": True,
        "verify_nbf": True,
    },
)

if await blocklist.contains(claims["jti"]):
    raise AuthError("token revoked")
```

Concrete TTL ranges (industry consensus, not from the RFC):

- **Access tokens** (used on every API call): **5–15 minutes**. Short enough that a leaked token isn't a long-term liability.
- **Refresh tokens** (used to mint new access tokens): hours to days, **opaque** (not JWT) and stored server-side so they can be revoked by deletion.
- **OIDC ID tokens** ([[JWT UC 03 - OIDC ID Tokens]]): minutes — they prove authentication, not authorisation.

The pattern (matches [[JWT UC 02 - OAuth Access Tokens]]):

1. User logs in → server issues a short-lived JWT access token + a long-lived opaque refresh token.
2. Client uses the access token until it expires (~10 min).
3. Client trades refresh token for a new access token at the `/token` endpoint.
4. Server can revoke the refresh token in its own DB at any time → next refresh attempt fails → access token expires within minutes → user is out.

RFC 7519 §4.1.4 explicitly allows clock-skew leeway:

> "Implementers MAY provide for some small leeway, usually no more than a few minutes, to account for clock skew."

Keep it small (30–60 seconds). Larger windows extend the lifetime of compromised tokens.

**Revocation lane.** When you genuinely need to invalidate before `exp`:

- Maintain a **blocklist keyed by `jti`** in Redis with TTL = remaining `exp` (so entries auto-expire). Check on every request.
- Or maintain a **per-user "min issued at"** timestamp; reject any token whose `iat` is older. Bumping the timestamp logs the user out everywhere.
- Refresh-token revocation alone is sufficient for "log out from this device" — the access token dies on its own within minutes.

```python
# Revocation check: reject any access token issued before the user's logout cutoff.
min_iat = await users.get_min_iat(claims["sub"])
if claims["iat"] < min_iat:
    raise AuthError("token issued before logout")
```

⚠️ `verify_exp=False` is a debugging flag that frequently survives into production. Audit your JWT decode call sites.

⚠️ JWTs have no built-in revocation. If your design says "we'll revoke them" and your storage is "just the JWT," you're conflating sessions with bearer tokens. Add the blocklist or use opaque session tokens — see [[JWT FN 01 - What Is a JWT]] on when JWTs are the wrong tool.

⚠️ Don't use `iat` *as* the expiry by computing `now - iat < N`. The token says when it was issued, not when it expires — use `exp`. Library defaults already do this; rolling your own is a chance to forget clock skew.

⚠️ RFC 7519 §4.1.7 requires `jti` uniqueness across issuers when issuers share a namespace: "if the application uses multiple issuers, collisions MUST be prevented." A UUIDv4 per token is the simplest answer.

Cross-references:

- The claims themselves: [[JWT FN 03 - Registered Claims]].
- Why "valid signature" doesn't mean "still authoritative": [[JWT FN 05 - Validating vs Verifying]].
- BCP overview: [[JWT SE 01 - RFC 8725 BCP]].
- Algorithm-level concerns: [[JWT SE 02 - Algorithm Confusion]].
- Library-level enforcement: [[JWT PR 01 - PyJWT]], [[JWT PR 03 - jsonwebtoken Node]].
- The OAuth refresh-token flow that this design assumes: [[JWT UC 02 - OAuth Access Tokens]].

💡 **Takeaway:** keep access-token `exp` short (minutes), enforce `exp`/`nbf`/`iat`/`jti`, and route long-lived state through revocable refresh tokens.
