# UC 03 — OIDC ID Tokens

🔑 **One-line: the ID Token is OpenID Connect's "the user just authenticated, here's who they are" assertion — a JWT for the *client*, not for any resource server.**

## Pattern

[OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) layers identity on top of OAuth 2.0. Where OAuth gives the client an *access token* to call resources ([[JWT UC 02 - OAuth Access Tokens]]), OIDC adds an *ID token* that says "this user authenticated, with these properties, just now."

§2: "The ID Token is a security token that contains Claims about the Authentication of an End-User by an Authorization Server."

The flow (Authorization Code Flow):

1. Relying Party (RP, the client app) redirects the user-agent to the OpenID Provider (OP) with `response_type=code`, `scope=openid ...`, `nonce=<random>`, `state=<random>`.
2. User authenticates at OP. OP redirects back with `?code=...&state=...`.
3. RP exchanges code at the token endpoint, gets back:

```json
{
  "access_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "...",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Ii4uLiJ9..."
}
```

4. RP validates the ID token (every claim, every signature), establishes a local session for the user. **The ID token is consumed by the RP. It is never sent back to the OP, and it is not used as a bearer token against APIs.**

## Token shape

```json
// header
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "op-2024-1"
}
```

```json
// payload — Authorization Code Flow
{
  "iss": "https://accounts.example.com",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "exp": 1714233600,
  "iat": 1714230000,
  "auth_time": 1714229987,
  "nonce": "n-0S6_WzA2Mj",
  "acr": "urn:mace:incommon:iap:silver",
  "amr": ["pwd", "mfa"],
  "azp": "s6BhdRkqt3",
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice Example"
}
```

```json
// payload — Hybrid / Implicit Flow adds hash claims
{
  "...": "...",
  "at_hash": "77QmUPtjPfzWtF2AnpK9RQ",
  "c_hash": "LDktKdoQak3Pk0cnXxCltA"
}
```

### Required claims (§2)

| Claim | Definition (verbatim from OIDC Core) |
|---|---|
| `iss` | "Issuer Identifier... case-sensitive URL using the https scheme... no query or fragment components." |
| `sub` | "A locally unique and never reassigned identifier within the Issuer for the End-User... MUST NOT exceed 255 ASCII characters in length." |
| `aud` | "Audience(s) that this ID Token is intended for. It MUST contain the OAuth 2.0 client_id of the Relying Party as an audience value." |
| `exp` | "Expiration time on or after which the ID Token MUST NOT be accepted by the RP." |
| `iat` | "Time at which the JWT was issued." |

### Conditional / optional

- **`auth_time`** — REQUIRED when `max_age` was requested or when `auth_time` was requested as an Essential Claim; otherwise OPTIONAL.
- **`nonce`** — passed through unmodified from the auth request to the ID token. RP **MUST** verify this when present, and **MUST** include it for Implicit/Hybrid flows. Mitigates replay.
- **`acr`** — Authentication Context Class Reference. String identifying the assurance level satisfied.
- **`amr`** — Authentication Methods References. JSON array of strings (`pwd`, `mfa`, `otp`, `hwk`, `swk`, `face`, `fpt`, `iris`, ...).
- **`azp`** — Authorized Party. "If present, it MUST contain the OAuth 2.0 Client ID of this party." Used when `aud` has multiple values.

### Hash claims (flow-specific)

- **`at_hash`** — REQUIRED in Implicit Flow when `id_token` is returned with `access_token`; OPTIONAL in Code Flow. "Base64url encoding of the left-most half of the hash of the octets of the ASCII representation of the access_token value." Algorithm matches `alg` (RS256 → SHA-256, take left 128 bits).
- **`c_hash`** — REQUIRED in Hybrid Flow when `id_token` is returned with `code`. Same construction over the `code` value.

These bind the ID token to the access token / authorization code so a man-in-the-middle can't swap them.

## Validation

OIDC Core §3.1.3.7 — the **complete** list. The RP **MUST** perform every step that applies:

1. **Decrypt** if the ID token is a JWE, using keys/algs registered.
2. **Issuer.** "The Issuer Identifier for the OpenID Provider... MUST exactly match the value of the `iss` (issuer) Claim."
3. **Audience.** "The Client MUST validate that the `aud` (audience) Claim contains its `client_id` value... The ID Token MUST be rejected if the ID Token does not list the Client as a valid audience."
4. **Multiple audiences.** If `aud` contains values not trusted by the client, the ID token MUST be rejected. If multiple audiences are present, `azp` SHOULD be present.
5. **`azp` check.** "If the `azp` (authorized party) Claim is present, the Client SHOULD verify that its `client_id` is the Claim Value."
6. **Signature.** "The Client MUST validate the signature of all other ID Tokens according to JWS [[JWT JO 02 - JWS]] using the algorithm specified in the JWT `alg` Header Parameter."
7. **Algorithm.** "The `alg` value SHOULD be the default of RS256 or the algorithm sent by the Client in the `id_token_signed_response_alg` parameter during Registration." Reject anything outside the registered list. **Reject `none` always.** See [[JWT SE 05 - Validation Pitfalls]].
8. **MAC keys.** If `alg` is HMAC-based (HS256/384/512), the key is "the octets of the UTF-8 representation of the `client_secret`." Asymmetric algs use the OP's JWKS ([[JWT JO 05 - JWK]]).
9. **Expiry.** "The current time MUST be before the time represented by the `exp` Claim."
10. **Issued-at sanity.** "The `iat` Claim can be used to reject tokens that were issued too far away from the current time."
11. **Nonce.** "If a nonce value was sent in the Authentication Request, a nonce Claim MUST be present and its value checked to verify it is the same value as the one sent." Reject on mismatch.
12. **`acr` check.** If the RP requested specific `acr` values, the asserted value MUST be appropriate.
13. **`auth_time` check.** If `max_age` was requested or `auth_time` is essential, "the Client SHOULD check the `auth_time` Claim value and request re-authentication if too much time has elapsed since the last End-User authentication."

For Implicit/Hybrid flows, additionally validate `at_hash` against the `access_token` and `c_hash` against the `code` (§3.2.2.9, §3.3.2.11).

See [[JWT FN 05 - Validating vs Verifying]] — signature verification is one step out of thirteen.

## ⚠️ Footguns

- **Treating ID tokens as API auth tokens.** The number-one OIDC mistake. The ID token's `aud` is the *client*, not your API. Sending the ID token in `Authorization: Bearer ...` against your backend is *not* authentication of the user — it's a stolen ID token being replayed. APIs use access tokens ([[JWT UC 02 - OAuth Access Tokens]]).
- **Skipping `nonce`.** The whole point of `nonce` is replay-attack defence. RPs that don't send / don't verify `nonce` for Implicit and Hybrid flows are vulnerable. RFC requires it.
- **Loose issuer compare.** `iss` is a URL, but you compare it as an exact string — no trailing-slash normalization, no scheme coercion, no case folding on the host.
- **`aud` array assumption.** `aud` can be a single string OR an array. `claims["aud"] == client_id` will silently fail when it's `["client_id", "other"]`. Always handle both shapes.
- **Trusting `email` for identity.** `sub` is the only identifier guaranteed to be stable per OIDC. `email` can change, can be unverified, can be reassigned at the user's email provider. Key on `(iss, sub)` pairs in your DB.
- **Confused with JWT access tokens.** They look similar — same JWT envelope, same `iss`/`sub`/`aud`/`exp`. Distinguish by `aud` (RP for ID tokens, RS for access tokens) and ideally `typ` (`at+jwt` for access tokens — see [[JWT SE 06 - Cross-JWT Confusion]]).
- **Skipping `at_hash` / `c_hash`.** In Implicit/Hybrid these bind the front-channel ID token to the back-channel access token / code. Skipping them re-opens substitution attacks the spec is closing.
- **HS256 with weak `client_secret`.** OIDC permits HMAC ID tokens keyed on `client_secret`. If you ever do that, the `client_secret` must be strong enough to be a key — see [[JWT SE 03 - Key Management]] and RFC 8725's "Ensure Cryptographic Keys Have Sufficient Entropy."
- **Re-using `state` for `nonce` (or vice versa).** They serve different purposes — `state` is CSRF defence on the redirect, `nonce` is replay defence on the token. Both required, both verified separately.

## Worked example: code flow validation in PyJWT

Outline using [[JWT PR 01 - PyJWT]]:

```python
import jwt
from jwt import PyJWKClient

jwks_client = PyJWKClient("https://accounts.example.com/.well-known/jwks.json")
signing_key = jwks_client.get_signing_key_from_jwt(id_token).key

claims = jwt.decode(
    id_token,
    key=signing_key,
    algorithms=["RS256"],
    audience=client_id,                       # step 3
    issuer="https://accounts.example.com",    # step 2
    options={"require": ["exp", "iat", "iss", "aud", "sub"]},
)

if claims.get("nonce") != expected_nonce:     # step 11
    raise SecurityError("nonce mismatch")

if claims.get("azp") and claims["azp"] != client_id:  # step 5
    raise SecurityError("azp mismatch")
```

Step 6 (signature) and steps 2, 3, 9, 10 are handled by `jwt.decode` when the right options are passed. The rest you do yourself. See [[JWT SE 05 - Validation Pitfalls]] for the cases where library defaults don't.

## Related

- [[JWT UC 01 - API Authentication]] — the ID token does *not* go here; the *access token* does
- [[JWT UC 02 - OAuth Access Tokens]] — same envelope, completely different consumer
- [[JWT UC 04 - Storage and Transport]] — ID tokens typically sit in the RP server-side session, not in the browser
- [[JWT FN 01 - What Is a JWT]], [[JWT FN 03 - Registered Claims]] — primitives
- [[JWT SE 06 - Cross-JWT Confusion]] — why `typ` and `aud` matter so much

💡 **Takeaway:** the ID token is for the *Relying Party* and asserts authentication; thirteen validation steps in OIDC Core §3.1.3.7 are non-negotiable; never use it as a bearer credential against an API — that's the access token's job.
