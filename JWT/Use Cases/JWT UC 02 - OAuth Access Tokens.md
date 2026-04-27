# UC 02 — OAuth Access Tokens

🔑 **One-line: an OAuth 2.0 access token formatted as a JWT — RFC 9068 nails down which claims must be there so resource servers can validate without phoning home.**

## Pattern

OAuth 2.0 ([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)) defines an *access token* as "a string representing an authorization issued to the client." The string format is opaque by spec — could be a random GUID backed by an introspection endpoint, could be a JWT.

[RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068) (JWT Profile for OAuth 2.0 Access Tokens) standardizes the JWT-shaped variant. Now resource servers can validate locally instead of calling `POST /oauth/introspect` on every request.

The flow:

1. Client obtains an authorization grant (auth code, client credentials, etc.) from the authorization server (AS).
2. Client exchanges the grant at the token endpoint:

```http
POST /oauth/token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=...&client_id=...&scope=read+write
```

3. AS responds (RFC 6749 §5.1):

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6ImF0K2p3dCIsImtpZCI6Im...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read write",
  "refresh_token": "8xLOxBtZp8"
}
```

4. Client sends the access token to the resource server (RS) per [[JWT UC 01 - API Authentication]] — `Authorization: Bearer ...`.
5. RS validates the JWT locally using the AS's public keys (JWKS — see [[JWT JO 05 - JWK]]).

## Token shape

RFC 9068 mandates `typ: at+jwt` in the **header** to distinguish access tokens from ID tokens and other JWTs ([[JWT SE 06 - Cross-JWT Confusion]]):

```json
// header
{
  "typ": "at+jwt",
  "alg": "RS256",
  "kid": "2024-key-1"
}
```

```json
// payload
{
  "iss": "https://as.example.com",
  "exp": 1714233600,
  "aud": "https://api.example.com",
  "sub": "user_8f2a91c3",
  "client_id": "myapp_web",
  "iat": 1714230000,
  "jti": "01HW8XK3VQ5R9N7M2P4T6Y8Z0A",
  "scope": "orders:read orders:write",
  "auth_time": 1714229000,
  "acr": "urn:mace:incommon:iap:silver",
  "amr": ["pwd", "mfa"],
  "groups": ["staff", "ops"],
  "roles": ["admin"],
  "entitlements": ["paid_tier"]
}
```

### Mandatory claims (RFC 9068 §2.2)

| Claim | Meaning |
|---|---|
| `iss` | URL of the issuing AS — exact-match string compare |
| `exp` | Expiration timestamp |
| `aud` | The RS this token is for (one or many) |
| `sub` | Resource owner, or the client itself for `client_credentials` |
| `client_id` | OAuth client that requested the token |
| `iat` | Issued-at timestamp |
| `jti` | Unique token id — useful for replay detection / revocation lists |

See [[JWT FN 03 - Registered Claims]] for the underlying RFC 7519 definitions.

### Authentication context (when a user authenticated)

- `auth_time` — when they logged in
- `acr` — Authentication Context Class Reference
- `amr` — methods used (`pwd`, `mfa`, `otp`, `hwk`, `swk`, ...)

### Authorization claims

- `scope` — space-delimited string from RFC 6749. RFC 9068: "All the individual scope strings in the 'scope' claim MUST have meaning for the resources indicated in the 'aud' claim."
- `groups` / `roles` / `entitlements` — optional, when the AS has that knowledge

## Validation

RFC 9068 §4 — the RS **MUST** perform the following on every request:

1. **Type check.** The `typ` header must be `at+jwt` (or `application/at+jwt`). Reject anything else — this is what stops an ID token from being smuggled in as an access token.
2. **Decrypt** if the token is a JWE per registration.
3. **Issuer.** `iss` must exactly match the expected AS identifier.
4. **Audience.** `aud` must contain this RS's identifier.
5. **Signature.** Verify per [[JWT JO 02 - JWS]] using a key from the AS's JWKS, matched by `kid`. **MUST reject `alg: none`.**
6. **Expiry.** Current time strictly before `exp`. `nbf` if present.
7. **Scope/roles.** Enforce that the requested operation is covered by `scope` / `roles` / `entitlements` — return `insufficient_scope` (403) per [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750) on mismatch.

See [[JWT FN 05 - Validating vs Verifying]] for why "verify" alone is not enough.

## ⚠️ Footguns

- **Forgetting `typ: at+jwt`.** Without explicit typing and a `typ` check, an ID token (which also has `iss`, `aud`, `sub`, signed by the same AS) can be passed off as an access token. RFC 9068 exists exactly because of this attack class.
- **Cross-resource replay.** A token issued for `aud: orders-api` accepted by `payments-api` because both share an issuer and nobody checks `aud`. RFC 9068: "Use distinct `aud` values for different resources to prevent cross-JWT confusion." See [[JWT SE 06 - Cross-JWT Confusion]].
- **Scope is a string, not an array.** RFC 6749: space-delimited, case-sensitive strings. Don't naively `JSON.parse` — split on `" "`.
- **Refresh tokens are not JWTs.** Don't conflate. Refresh tokens are usually opaque, server-side state, used only at the token endpoint.
- **`client_credentials` grants have no user.** `sub` will typically equal `client_id`, and `auth_time`/`acr`/`amr` are absent. Code that demands `auth_time` will break for machine-to-machine.
- **Client-modified `sub`.** The AS MUST prevent clients from injecting `sub`. Naive AS implementations that pass through user-supplied identity claims on `client_credentials` are a privilege-escalation primitive.
- **Long-lived tokens.** RFC 6750 §5.3 wants ≤1 hour. Pair short access tokens with refresh tokens; don't issue 30-day access tokens to "save round-trips."
- **HMAC across the AS↔RS boundary.** HS256 means the AS shares its signing secret with every RS. Use RS256/ES256 ([[JWT AL 02 - RSA Signatures]], [[JWT AL 03 - ECDSA]]) and JWKS-distributed public keys.

## Versus opaque tokens

| Aspect | JWT access token (RFC 9068) | Opaque + introspection (RFC 7662) |
|---|---|---|
| RS validation cost | Local signature verify | Network call to AS per request |
| Revocation | Hard — wait for `exp` or maintain `jti` blacklist | Easy — AS just stops returning `active: true` |
| Token size | Larger (claims inline) | Small random string |
| AS load | Once at issuance + JWKS fetches | Every request hits introspection |

JWT access tokens trade revocation crispness for AS-decoupled validation. See [[JWT SE 04 - Expiration and Revocation]].

## Related

- [[JWT UC 01 - API Authentication]] — the transport layer (Bearer header) is the same
- [[JWT UC 03 - OIDC ID Tokens]] — superficially similar shape, completely different audience and purpose
- [[JWT UC 04 - Storage and Transport]] — where to keep the access token in a browser

💡 **Takeaway:** an RFC 9068 access token is a JWT with `typ: at+jwt`, the seven required claims (`iss`, `exp`, `aud`, `sub`, `client_id`, `iat`, `jti`), and a `scope` claim — validated locally by the resource server with the issuer's public key, eliminating the per-request introspection round-trip.
