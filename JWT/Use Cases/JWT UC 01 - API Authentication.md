# UC 01 — API Authentication

🔑 **One-line: Bearer JWTs in `Authorization` headers turn stateless HTTP requests into authenticated ones — at the cost of "valid until expiry, no take-backs."**

## Pattern

The canonical flow per [jwt.io/introduction](https://jwt.io/introduction) and [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750):

1. **Login.** Client posts credentials to `POST /auth/login`. Server verifies, signs a JWT, returns it in the response body.
2. **Store.** Client stashes the token (in memory, cookie, or — at its peril — localStorage; see [[JWT UC 04 - Storage and Transport]]).
3. **Request.** Every protected call carries the token:

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

4. **Validate.** Server verifies signature against its public key (or HMAC secret), checks claims, runs the handler. **No DB lookup for the token itself.**

The grammar is fixed by [RFC 6750 §2.1](https://datatracker.ietf.org/doc/html/rfc6750#section-2.1):

```
credentials = "Bearer" 1*SP b64token
b64token    = 1*( ALPHA / DIGIT / "-" / "." / "_" / "~" / "+" / "/" ) *"="
```

Resource servers **MUST** support the `Authorization` header. Form-body (`access_token=...` with `Content-Type: application/x-www-form-urlencoded`) is permitted but second-class. URI query (`?access_token=...`) is **SHOULD NOT** — it leaks into logs, browser history, and `Referer` headers.

## Token shape

A bare-minimum API auth JWT — see [[JWT FN 03 - Registered Claims]]:

```json
{
  "iss": "https://api.example.com",
  "sub": "user_8f2a91c3",
  "aud": "https://api.example.com",
  "exp": 1714233600,
  "iat": 1714230000,
  "jti": "01HW8XK3VQ5R9N7M2P4T6Y8Z0A",
  "scope": "orders:read orders:write"
}
```

Signed with RS256 ([[JWT AL 02 - RSA Signatures]]) or ES256 ([[JWT AL 03 - ECDSA]]) so the API can verify with a public key fetched from a JWKS ([[JWT JO 05 - JWK]]) — never HS256 across service boundaries.

## Validation

Per [[JWT FN 05 - Validating vs Verifying]], the resource server **MUST**:

1. Parse the header, reject `alg: none` outright, reject any algorithm not on the allowlist.
2. Resolve the verification key (JWKS by `kid`, or pinned key) — see [[JWT SE 03 - Key Management]].
3. Verify the JWS signature ([[JWT JO 02 - JWS]]).
4. `iss` matches expected issuer, exactly. String compare, no normalization.
5. `aud` contains this API's identifier.
6. `exp` is in the future, `nbf`/`iat` not in the future (allow ~60s clock skew).
7. Token isn't on a revocation list if you keep one (see [[JWT SE 04 - Expiration and Revocation]]).

On failure, return per [RFC 6750 §3](https://datatracker.ietf.org/doc/html/rfc6750#section-3):

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api.example.com",
                  error="invalid_token",
                  error_description="The access token expired"
```

Three standard error codes:
- **`invalid_request`** → 400 — malformed, missing param, multiple methods used
- **`invalid_token`** → 401 — expired, revoked, bad signature
- **`insufficient_scope`** → 403 — token valid but scope doesn't cover this resource

## ⚠️ Footguns

- **TLS is non-negotiable.** RFC 6750: "Clients MUST always use TLS (https) or equivalent transport security." A bearer token over plaintext HTTP is a free credential for any sniffer on the path.
- **Stateless = unrevocable.** Until `exp`, the token is good. Logout button alone changes nothing — see [[JWT SE 04 - Expiration and Revocation]].
- **Don't put tokens in query strings.** They land in access logs, CDN logs, browser history, `Referer` headers sent to third parties.
- **Header size limits.** Most servers cap headers at 8 KB. Stuffing the entire user profile into the JWT will eventually wedge requests behind nginx/ALB defaults.
- **Confused deputy.** If two APIs both accept the same `iss` but you forget to check `aud`, a token for service A authenticates against service B. Always validate `aud`. See [[JWT SE 06 - Cross-JWT Confusion]].
- **Cookies sent in cleartext.** RFC 6750 §5.2: "Bearer tokens MUST NOT be stored in cookies that can be sent in the clear." `Secure` flag, always.
- **Short lifetimes.** RFC 6750 §5.3: "Token servers SHOULD issue short-lived (one hour or less) bearer tokens." Pair with refresh tokens, not 30-day access tokens.

## Library quickstarts

Verifying an incoming token in Python ([[JWT PR 01 - PyJWT]]):

```python
import jwt
claims = jwt.decode(
    token,
    key=public_key,
    algorithms=["RS256"],
    audience="https://api.example.com",
    issuer="https://api.example.com",
)
```

In Node ([[JWT PR 03 - jsonwebtoken Node]]):

```js
jwt.verify(token, publicKey, {
  algorithms: ["RS256"],
  audience: "https://api.example.com",
  issuer: "https://api.example.com",
});
```

See [[JWT SE 05 - Validation Pitfalls]] for everything these one-liners can still get wrong (missing `algorithms`, missing `audience`, swallowing errors).

## Related use cases

- [[JWT UC 02 - OAuth Access Tokens]] — when the API auth token is an OAuth access token with scopes
- [[JWT UC 03 - OIDC ID Tokens]] — ID tokens are *not* API auth tokens; do not confuse the two
- [[JWT UC 04 - Storage and Transport]] — where to actually keep the thing on the client

💡 **Takeaway:** API auth with JWTs is a Bearer token in the `Authorization` header, validated locally with no DB hit — fast and simple, but every token is a self-contained capability until it expires, so keep lifetimes short and audiences strict.
