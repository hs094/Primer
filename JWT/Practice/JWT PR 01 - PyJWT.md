# PR 01 — PyJWT

🔑 **One-line: the de facto Python library for encoding, decoding, and verifying JWTs (RFC 7519) with first-class JWKS support.**

PyJWT covers symmetric ([[JWT AL 01 - HMAC]]), RSA ([[JWT AL 02 - RSA Signatures]]), ECDSA ([[JWT AL 03 - ECDSA]]), and Ed25519 ([[JWT AL 04 - EdDSA]]) signing — i.e. the algorithms defined by [[JWT JO 06 - JWA]] for [[JWT JO 02 - JWS]]. See also [[JWT PR 02 - joserfc Python]], [[JWT PR 03 - jsonwebtoken Node]], [[JWT PR 04 - jwt.io Debugger]].

## Install

```bash
# Pure HMAC only
pip install pyjwt

# With RSA / ECDSA / EdDSA support (needed for RS256, ES256, EdDSA, JWKS)
pip install "pyjwt[crypto]"
```

## Encode / sign

HS256 ([[JWT AL 01 - HMAC]]):

```python
import jwt

encoded = jwt.encode({"some": "payload"}, "secret", algorithm="HS256")
```

RS256 ([[JWT AL 02 - RSA Signatures]]):

```python
encoded = jwt.encode({"some": "payload"}, private_key, algorithm="RS256")
```

ES256 ([[JWT AL 03 - ECDSA]]):

```python
encoded = jwt.encode({"some": "payload"}, private_key, algorithm="ES256")
```

EdDSA ([[JWT AL 04 - EdDSA]]):

```python
encoded = jwt.encode({"some": "payload"}, private_key, algorithm="EdDSA")
```

Custom header (e.g. `kid` for [[JWT JO 05 - JWK]] lookup):

```python
jwt.encode(
    {"some": "payload"},
    "secret",
    algorithm="HS256",
    headers={"kid": "230498151c214b788dd97f22b85410a5"},
)
```

## Decode / verify

The signature, expiration, audience, issuer all get checked. See [[JWT FN 05 - Validating vs Verifying]].

```python
jwt.decode(encoded, "secret", algorithms=["HS256"])
# {'some': 'payload'}
```

With audience and issuer ([[JWT FN 03 - Registered Claims]]):

```python
jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],
    audience="urn:foo",
    issuer="urn:foo",
)
```

Clock skew leeway:

```python
jwt.decode(token, "secret", leeway=5, algorithms=["HS256"])
```

## Registered claims

```python
from datetime import datetime, timezone, timedelta
import uuid

payload = {
    "iss": "urn:my-issuer",
    "sub": "user-123",
    "aud": ["urn:foo", "urn:bar"],
    "exp": datetime.now(tz=timezone.utc) + timedelta(hours=1),
    "nbf": datetime.now(tz=timezone.utc),
    "iat": datetime.now(tz=timezone.utc),
    "jti": str(uuid.uuid4()),
}
token = jwt.encode(payload, "secret", algorithm="HS256")
```

## JWKS client (for OIDC, Auth0, Cognito, etc.)

The killer feature for verifying [[JWT UC 03 - OIDC ID Tokens]] — fetches and caches the issuer's [[JWT JO 05 - JWK]] set, picks the right key by `kid`.

```python
import jwt
from jwt import PyJWKClient

url = "https://example.com/.well-known/jwks.json"
jwks_client = PyJWKClient(url)

signing_key = jwks_client.get_signing_key_from_jwt(token)

data = jwt.decode(
    token,
    signing_key,
    audience="api",
    algorithms=["RS256"],
)
```

`PyJWKClient` does two-tier caching: the JWK set itself (`cache_jwk_set`, `lifespan=300`) and individual keys (`cache_keys`, `max_cached_keys=16`).

## Inspecting without verification

Useful for reading `kid` before key lookup — but never trust the result. See [[JWT SE 05 - Validation Pitfalls]].

```python
header = jwt.get_unverified_header(token)
# {'alg': 'RS256', 'kid': '...'}

payload = jwt.decode(token, options={"verify_signature": False})
```

## Error types

```python
try:
    jwt.decode(token, key, algorithms=["RS256"], audience="api")
except jwt.ExpiredSignatureError:
    ...                              # exp passed
except jwt.InvalidAudienceError:
    ...                              # aud mismatch
except jwt.InvalidIssuerError:
    ...                              # iss mismatch
except jwt.InvalidSignatureError:
    ...                              # sig didn't verify
except jwt.MissingRequiredClaimError:
    ...                              # required claim absent
except jwt.InvalidTokenError:
    ...                              # base class — catches all of the above
```

`PyJWKClientError` covers JWKS fetch/parse failures.

## Footguns

⚠️ **Always pass `algorithms=[...]` explicitly.** Never derive it from the token header — that's the classic [[JWT SE 02 - Algorithm Confusion]] attack (RFC 8725). Pin the exact algorithm(s) you accept.

⚠️ **`pip install pyjwt` alone won't do RS256/ES256/EdDSA.** You need `pyjwt[crypto]` for the `cryptography` backend. Otherwise you'll see `NotImplementedError` or unsupported-algorithm errors.

⚠️ **`get_unverified_header` / `verify_signature=False` are inspection-only.** Anyone can forge the header — never gate authorization on unverified claims.

⚠️ **HMAC + asymmetric public key = RCE-grade bug.** If a verifier accepts both `HS256` and `RS256` and the attacker flips `alg` to `HS256`, the public key gets used as the HMAC secret. Mitigation: pin one algorithm class. See [[JWT SE 03 - Key Management]].

⚠️ **`exp` and `nbf` use `NumericDate` (seconds-since-epoch).** Passing a naive `datetime` works but always prefer `datetime.now(tz=timezone.utc)`.

💡 **Takeaway:** `pip install "pyjwt[crypto]"`, always pin `algorithms=[...]`, use `PyJWKClient` for OIDC verification — and never trust `get_unverified_header` for anything but `kid` lookup.
