# FN 03 — Registered Claims

🔑 **Seven IANA-registered claim names every JWT producer and consumer should already know: `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`. All optional, all three letters, all interoperable.**

## Why Registered Names Exist (RFC 7519 §4.1)

> The following Claim Names are registered in the IANA "JSON Web Token Claims" registry [...] None of the claims defined below are intended to be mandatory to use or implement in all cases, but rather they provide a starting point for a set of useful, interoperable claims.

Three letters keep tokens compact. Use them whenever the semantics fit; don't reinvent them with custom names.

## Cheat Sheet

```json
{
  "iss": "https://auth.example.com",
  "sub": "user-42",
  "aud": "api.example.com",
  "exp": 1735689600,
  "nbf": 1735689000,
  "iat": 1735689000,
  "jti": "5f8d0c4e-7c1f-4f6a-9b2e-3a8e9b8e9b8e"
}
```

`exp`, `nbf`, `iat` are **NumericDate** — seconds since Unix epoch (UTC).

## `iss` — Issuer

> The "iss" (issuer) claim identifies the principal that issued the JWT.

- StringOrURI; case-sensitive.
- Verifier MUST pin the expected issuer and reject mismatches.

```python
if payload["iss"] != "https://auth.example.com":
    raise InvalidIssuer
```

⚠️ **Don't trust `iss` to pick the verification key without bounds.** Algorithm confusion attacks ride on dynamic `iss` lookups. See [[JWT SE 02 - Algorithm Confusion]].

## `sub` — Subject

> The "sub" (subject) claim identifies the principal that is the subject of the JWT. The Claims in a JWT are normally statements about the subject. The subject value MUST either be scoped to be locally unique in the context of the issuer or be globally unique.

Usually a stable user id. Avoid email — emails change.

```json
{ "sub": "auth0|6651f0d2c1a0f4e1a1b2c3d4" }
```

## `aud` — Audience

> The "aud" (audience) claim identifies the recipients that the JWT is intended for. Each principal intended to process the JWT MUST identify itself with a value in the audience claim. If the principal processing the claim does not identify itself with a value in the "aud" claim when this claim is present, then the JWT MUST be rejected.

- May be a string or array of strings.
- Each verifier checks its own identifier appears.

```json
{ "aud": ["api.example.com", "billing.example.com"] }
```

⚠️ **Forgetting `aud` checks lets a token issued for service A be replayed against service B.** This is the most-skipped check in homegrown verifiers. See [[JWT SE 05 - Validation Pitfalls]].

## `exp` — Expiration Time

> The "exp" (expiration time) claim identifies the expiration time on or after which the JWT MUST NOT be accepted for processing. The processing of the "exp" claim requires that the current date/time MUST be before the expiration date/time listed in the "exp" claim. Implementers MAY provide for some small leeway, usually no more than a few minutes, to account for clock skew.

NumericDate. Keep lifetimes short for access tokens.

```python
if now >= payload["exp"]:
    raise TokenExpired
```

⚠️ **Long-lived JWTs can't be revoked.** Pair short `exp` with refresh tokens or a deny-list. See [[JWT SE 04 - Expiration and Revocation]].

## `nbf` — Not Before

> The "nbf" (not before) claim identifies the time before which the JWT MUST NOT be accepted for processing. The processing of the "nbf" claim requires that the current date/time MUST be after or equal to the not-before date/time listed in the "nbf" claim. Implementers MAY provide for some small leeway, usually no more than a few minutes, to account for clock skew.

Useful for tokens that activate at a future time (e.g., scheduled access).

```json
{ "nbf": 1735689600 }
```

## `iat` — Issued At

> The "iat" (issued at) claim identifies the time at which the JWT was issued. This claim can be used to determine the age of the JWT.

Used for:

- Computing token age.
- Rejecting tokens issued before a "password changed at" timestamp (forced logout).

```python
if payload["iat"] < user.password_changed_at:
    raise TokenInvalidated
```

## `jti` — JWT ID

> The "jti" (JWT ID) claim provides a unique identifier for the JWT. The identifier value MUST be assigned in a manner that ensures that there is a negligible probability that the same value will be accidentally assigned to a different data object; if the application uses multiple issuers, collisions MUST be prevented among values produced by different issuers as well. The "jti" claim can be used to prevent the JWT from being replayed.

Use a UUIDv4 or a strong random string. Persist used `jti`s for the token's lifetime when replay protection matters.

```json
{ "jti": "5f8d0c4e-7c1f-4f6a-9b2e-3a8e9b8e9b8e" }
```

## Validation Order (Pragmatic)

1. Verify signature ([[JWT FN 05 - Validating vs Verifying]]).
2. `iss` — expected issuer.
3. `aud` — this service is in audience.
4. `exp` — not expired.
5. `nbf` — already valid.
6. `iat` — sane (not in the future, not before user invalidation).
7. `jti` — not in replay deny-list (if applicable).

## With PyJWT

```python
import jwt

claims = jwt.decode(
    token,
    key=public_key,
    algorithms=["RS256"],
    issuer="https://auth.example.com",
    audience="api.example.com",
    leeway=30,
)
```

PyJWT validates `exp`, `nbf`, `iat`, `iss`, `aud` automatically when the parameters are passed. See [[JWT PR 01 - PyJWT]].

## All Optional, But...

RFC says none are mandatory. In practice:

- `iss`, `aud`, `exp` should be considered required for any auth use case.
- `sub` should be present whenever the token represents a user.
- `iat` is cheap and helps debugging.
- `nbf`, `jti` are situational.

This minimum is also what [[JWT SE 01 - RFC 8725 BCP]] expects.

💡 **Takeaway:** Seven three-letter claims cover identity, audience, lifetime, and uniqueness. Pin `iss`, check `aud`, enforce `exp` — anything less is broken.
