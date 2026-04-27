# FN 04 — Public and Private Claims

🔑 **Beyond the seven registered names: public claims live in a global namespace (IANA-registered or URI-scoped); private claims are local agreements between issuer and consumer. Pick wrong and your tokens collide with someone else's.**

## The Three-Tier Naming Model (RFC 7519)

| Tier | Source | Collision risk | Example |
|------|--------|----------------|---------|
| Registered (§4.1) | IANA "JSON Web Token Claims" registry | None | `iss`, `aud`, `exp` |
| Public (§4.2) | IANA-registered *or* collision-resistant URI | None if scoped properly | `https://example.com/is_root` |
| Private (§4.3) | Mutual agreement between parties | High — handle with care | `tenant_id`, `role` |

Registered claims are covered in [[JWT FN 03 - Registered Claims]]. This note covers the other two.

## Public Claim Names (RFC 7519 §4.2)

> Claim Names can be defined at will by those using JWTs. However, in order to prevent collisions, any new Claim Name should either be registered in the IANA "JSON Web Token Claims" registry [...] or be a Public Name: a value that contains a Collision-Resistant Name. In each case, the definer of the name or value needs to take reasonable precautions to make sure they are in control of the part of the namespace they use to define the Claim Name.

Two routes to a public claim:

### Route 1 — Register with IANA

If the claim is broadly useful, submit it to the JWT Claims registry. Examples already there: `email`, `name`, `preferred_username`, `scope`, `azp`, `nonce` (mostly from OIDC).

### Route 2 — Use a Collision-Resistant Name

Namespace the claim under a URI you control:

```json
{
  "https://example.com/is_root": true,
  "https://acme.io/claims/billing_tier": "pro"
}
```

The URI doesn't have to resolve. It just has to be unambiguously yours.

## Private Claim Names (RFC 7519 §4.3)

> A producer and consumer of a JWT MAY agree to use Claim Names that are Private Names: names that are not Registered Claim Names or Public Claim Names. Unlike Public Claim Names, Private Claim Names are subject to collision and should be used with caution.

Plain unscoped strings, by mutual agreement:

```json
{
  "tenant_id": "acme-corp",
  "role": "admin",
  "feature_flags": ["beta_ui", "new_billing"]
}
```

Fast and ergonomic — fine for closed systems where the issuer and verifier are the same team.

⚠️ **Private claims collide silently.** If a downstream library, gateway, or future standard introduces `role` with different semantics, you have a bug that ships with no error.

## When to Use Which

```text
Issuer == Verifier (same team, same product) ?
   yes -> private claims, short names, ship it
   no  -> tokens cross trust boundaries ?
            yes -> public (URI-namespaced or IANA-registered)
            no  -> still safer to namespace
```

OIDC provides a useful pattern: registered claims (`sub`, `aud`, `exp`) plus IANA-public claims (`email`, `name`) plus URI-namespaced custom claims (e.g., `https://your-app.com/roles`). See [[JWT UC 03 - OIDC ID Tokens]].

## Auth0 / Okta Pattern

Auth0 forbids unrecognized top-level claims in ID tokens; custom claims must be namespaced:

```json
{
  "sub": "auth0|abc123",
  "https://myapp.example.com/roles": ["admin", "billing"],
  "https://myapp.example.com/tenant": "acme"
}
```

Verbose, but no IdP-version upgrade can ever stomp on you.

## Worked Example: Same Token, Three Tiers

```json
{
  "iss": "https://auth.example.com",
  "sub": "user-42",
  "aud": "api.example.com",
  "exp": 1735689600,

  "email": "alice@example.com",

  "https://example.com/department": "engineering",

  "internal_user_id": 90210
}
```

- Registered: `iss`, `sub`, `aud`, `exp`.
- Public (IANA): `email`.
- Public (URI): `https://example.com/department`.
- Private: `internal_user_id`.

## IANA Registry Lookup

The current registry: <https://www.iana.org/assignments/jwt/jwt.xhtml>. Check before defining a new short name — `scope`, `client_id`, `auth_time`, and many OIDC claims are already taken.

## Picking Names: Checklist

- Does a registered claim already cover this? → use it.
- Is the name listed in the IANA JWT registry? → public, OK to use.
- Does the token leave my trust boundary? → namespace it (URI public).
- Is it issuer-internal only? → private name acceptable, but document the agreement.

⚠️ **Don't shadow registered claims.** Never put your own meaning behind `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, or `jti`. Verifiers will treat them per RFC 7519 regardless.

⚠️ **Don't dump arbitrary user input into claims.** Anything an attacker controls can land in your token unless you sanitize it on the way in. See [[JWT SE 05 - Validation Pitfalls]].

## Code: Reading Custom Claims

```python
import jwt

claims = jwt.decode(token, key=pub, algorithms=["RS256"], audience="api")
roles: list[str] = claims.get("https://myapp.example.com/roles", [])
tenant: str | None = claims.get("https://myapp.example.com/tenant")
```

A `dict.get` with a default beats a `KeyError` on optional claims.

💡 **Takeaway:** Registered first, IANA-public second, URI-namespaced public third, private only inside one trust boundary. Namespace before someone else does.
