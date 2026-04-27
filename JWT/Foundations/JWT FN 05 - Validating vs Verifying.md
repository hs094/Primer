# FN 05 — Validating vs Verifying

🔑 **Verifying answers "is the signature good?" Validating answers "do the claims permit this request?" Both must pass. One without the other is a vulnerability.**

## Two Different Questions

| Step | Asks | Fails when... |
|------|------|---------------|
| **Verify (signature)** | Did the holder of the signing key produce this exact byte string? | Tampered payload, wrong key, wrong alg, `none` accepted |
| **Validate (claims)** | Are the claims acceptable to *this* service *right now*? | Wrong issuer, wrong audience, expired, not-yet-valid, replayed |

Either step alone is insufficient:

- Signature OK but `aud` wrong → token meant for another service. Reject.
- Claims look right but signature bad → forged. Reject.

## RFC 7519 §7.2 — Validation Steps

The RFC walks through both. Abridged:

> 1. Verify that the JWT contains at least one period ('.') character.
> 2. Let the Encoded JOSE Header be the portion of the JWT before the first period.
> 3. Base64url decode [...] following the restriction that no line breaks, whitespace, or other additional characters have been used.
> 4. Verify that the resulting octet sequence is a UTF-8-encoded representation of a completely valid JSON object [...].
> 5. Verify that the resulting JOSE Header includes only parameters and values whose syntax and semantics are both understood and supported [...].
> 6. Determine whether the JWT is a JWS or a JWE [...].
> 7. [...] follow the JWS validation rules [or JWE rules].

Then closes with:

> Finally, note that it is an application decision which algorithms may be used in a given context. Even if a JWT can be successfully validated, unless the algorithms used in the JWT are acceptable to the application, it SHOULD reject the JWT.

## Verify First, Then Validate

Order matters. If you read claims out of an unverified token to *decide how to verify it*, you've handed control to the attacker.

```python
import jwt

# WRONG — reads payload before verifying
unverified = jwt.decode(token, options={"verify_signature": False})
key = lookup_key(unverified["iss"])  # attacker controls iss
jwt.decode(token, key=key, algorithms=["RS256"])
```

```python
# RIGHT — pin trust before reading anything
EXPECTED_ISS = "https://auth.example.com"
EXPECTED_AUD = "api.example.com"
ALLOWED_ALGS = ["RS256"]

claims = jwt.decode(
    token,
    key=public_key_for(EXPECTED_ISS),
    algorithms=ALLOWED_ALGS,
    issuer=EXPECTED_ISS,
    audience=EXPECTED_AUD,
)
```

Pin `algorithms` to a specific list. Pin `issuer`. Pin `audience`. Then read the claims.

## What "Verify" Actually Does

For a JWS-protected JWT (the common case):

1. Split into `header.payload.signature`.
2. Base64url-decode header → JSON. Read `alg`, `kid`.
3. Confirm `alg` is in your allowlist.
4. Resolve the key (by `kid`, from a [[JWT JO 05 - JWK]] set, or a static secret).
5. Recompute the signature over `header_b64 + "." + payload_b64`.
6. Constant-time compare to the received signature.

Details by algorithm: [[JWT AL 01 - HMAC]] and [[JWT JO 06 - JWA]].

## What "Validate" Actually Does

After verification succeeds, walk the claims:

```python
def validate(claims: dict, *, now: int, expected_iss: str, expected_aud: str) -> None:
    if claims.get("iss") != expected_iss:
        raise InvalidIssuer
    aud = claims.get("aud")
    auds = aud if isinstance(aud, list) else [aud]
    if expected_aud not in auds:
        raise InvalidAudience
    if "exp" in claims and now >= claims["exp"]:
        raise TokenExpired
    if "nbf" in claims and now < claims["nbf"]:
        raise NotYetValid
```

Most JWT libraries do this for you when you pass `issuer=`, `audience=`, and have `exp`/`nbf` in the token. See [[JWT PR 01 - PyJWT]].

## Common Failure Modes

⚠️ **Trusting the `alg` header.** The token tells you what algorithm to use; the attacker can change it. Always pin to an allowlist. Drives the `none` and HS/RS confusion attacks: [[JWT AL 05 - none Algorithm]], [[JWT SE 02 - Algorithm Confusion]].

⚠️ **Skipping `aud`.** Tokens issued for one service get replayed at another. The most common gap in hand-rolled middleware.

⚠️ **No clock-skew leeway.** Distributed systems have drifting clocks. Allow ~30–60 seconds of leeway on `exp` and `nbf`.

⚠️ **`exp` only, no revocation path.** Long-lived tokens can't be invalidated. Pair short `exp` with refresh + deny-list. See [[JWT SE 04 - Expiration and Revocation]].

⚠️ **Decoding before verifying.** Even for "logging the user id," resist. Verify first.

## Practical Verification Recipe

```python
import jwt
from jwt import PyJWKClient

JWKS = PyJWKClient("https://auth.example.com/.well-known/jwks.json")

def verify_and_validate(token: str) -> dict:
    signing_key = JWKS.get_signing_key_from_jwt(token).key
    return jwt.decode(
        token,
        key=signing_key,
        algorithms=["RS256"],
        issuer="https://auth.example.com",
        audience="api.example.com",
        leeway=30,
        options={"require": ["exp", "iss", "aud"]},
    )
```

This single call:

- Pulls the right public key from JWKS by `kid`.
- Pins the algorithm.
- Verifies signature.
- Validates `iss`, `aud`, `exp`, `nbf`, `iat`.
- Errors if required claims are missing.

## Separation of Concerns

Keep the two steps in different functions if you build it yourself:

```python
def verify_signature(token: str) -> dict: ...
def validate_claims(claims: dict, *, ctx: RequestCtx) -> None: ...
```

Verification is a cryptographic concern (keys, algorithms). Validation is a business concern (who, when, for what). Mixing them is how bugs hide.

For the broader threat model, read [[JWT SE 01 - RFC 8725 BCP]] alongside [[JWT SE 05 - Validation Pitfalls]].

💡 **Takeaway:** Verify the signature with a pinned algorithm and key, then validate the claims with pinned issuer and audience. Both, in that order, every time.
