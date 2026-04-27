# PR 02 — joserfc (Python)

🔑 **One-line: the modern, RFC-faithful Python JOSE library — full JWS/JWE/JWK/JWA/JWT support, designed as the successor to the unmaintained `python-jose`.**

Where [[JWT PR 01 - PyJWT]] only covers [[JWT JO 02 - JWS]] / JWT, joserfc also implements JWE (encrypted tokens) and full [[JWT JO 05 - JWK]] / JWK Set machinery. Spun out of Authlib with a JOSE-only API surface. See also [[JWT PR 03 - jsonwebtoken Node]], [[JWT PR 04 - jwt.io Debugger]].

## Install

```bash
pip install joserfc
```

## RFC coverage

| RFC | Spec | Module |
|---|---|---|
| 7515 | [[JWT JO 02 - JWS]] | `joserfc.jws` |
| 7516 | JWE | `joserfc.jwe` |
| 7517 | [[JWT JO 05 - JWK]] | `joserfc.jwk` |
| 7518 | [[JWT JO 06 - JWA]] | algorithms across modules |
| 7519 | JWT | `joserfc.jwt` |

Plus draft specs for ECDH-1PU and modern algorithms ([[JWT AL 04 - EdDSA]] Ed25519/Ed448).

## Encode / sign

Symmetric ([[JWT AL 01 - HMAC]]):

```python
from joserfc import jwt
from joserfc.jwk import OctKey

key = OctKey.import_key("your-256-bit-secret")

encoded = jwt.encode(
    {"alg": "HS256"},                    # header
    {"sub": "user-123", "iss": "me"},    # claims
    key,
)
```

RSA ([[JWT AL 02 - RSA Signatures]]):

```python
from joserfc.jwk import RSAKey

key = RSAKey.import_key(open("private.pem").read())
encoded = jwt.encode({"alg": "RS256"}, {"sub": "u1"}, key)
```

EdDSA ([[JWT AL 04 - EdDSA]]):

```python
from joserfc.jwk import OKPKey

key = OKPKey.generate_key("Ed25519")
encoded = jwt.encode({"alg": "EdDSA"}, {"sub": "u1"}, key)
```

Generic `jwk.import_key` infers the key type:

```python
from joserfc import jwk

key = jwk.import_key("your-secret-key", "oct")
encoded = jwt.encode({"alg": "HS256"}, {"k": "value"}, key)
```

## Decode / verify

```python
from joserfc import jwt

token = jwt.decode(encoded, key)
token.header   # {'alg': 'HS256', 'typ': 'JWT'}
token.claims   # {'sub': 'user-123', 'iss': 'me'}
```

Claims validation is a separate step ([[JWT FN 05 - Validating vs Verifying]]):

```python
from joserfc.jwt import JWTClaimsRegistry

claims_registry = JWTClaimsRegistry(
    iss={"essential": True, "value": "me"},
    aud={"essential": True, "value": "api"},
    exp={"essential": True},
)
claims_registry.validate(token.claims)
```

## Algorithm allowlist

joserfc forces you to declare allowed algorithms — no implicit acceptance. Defends against [[JWT SE 02 - Algorithm Confusion]].

```python
token = jwt.decode(encoded, key, algorithms=["HS256"])
```

## JWK and JWK Set

```python
from joserfc.jwk import KeySet, RSAKey

# Build a key set
ks = KeySet([
    RSAKey.generate_key(2048, parameters={"kid": "key-1"}),
    RSAKey.generate_key(2048, parameters={"kid": "key-2"}),
])

# Export as a public JWKS for /.well-known/jwks.json
public_jwks = ks.as_dict(private=False)
```

```python
# Verify with a key set — joserfc picks the right key by `kid`
token = jwt.decode(encoded, ks, algorithms=["RS256"])
```

## JWE (encrypted tokens)

```python
from joserfc import jwe
from joserfc.jwk import RSAKey

key = RSAKey.generate_key(2048)

protected = {"alg": "RSA-OAEP", "enc": "A256GCM"}
ciphertext = jwe.encrypt_compact(protected, b"sensitive payload", key)

decrypted = jwe.decrypt_compact(ciphertext, key)
decrypted.plaintext   # b'sensitive payload'
```

## Patterns

- **Migrating from `python-jose`:** the project is unmaintained and has CVEs — joserfc is the recommended drop-in. Its docs include a migration guide.
- **Migrating from PyJWT:** joserfc's API is more verbose because it separates header / claims / key explicitly. Worth it when you also need JWE.
- **Building an issuer:** use `KeySet` so you can rotate without breaking existing tokens — old `kid` stays valid until its tokens expire. See [[JWT SE 03 - Key Management]].

## Footguns

⚠️ **`python-jose` is not joserfc.** They're different libraries. `python-jose` (PyPI: `python-jose`) is unmaintained; joserfc (PyPI: `joserfc`) is the active successor. Don't confuse import paths.

⚠️ **Claims aren't validated by `jwt.decode`.** It only verifies the signature. You must run `JWTClaimsRegistry.validate(...)` (or your own checks) for `exp`, `iss`, `aud`. This trips people up coming from PyJWT.

⚠️ **Use `OctKey` not raw bytes for symmetric algs.** Passing a string sometimes works but explicit `OctKey.import_key(...)` is the supported path.

⚠️ **JWE ≠ signed.** Encryption hides content but doesn't authenticate the sender alone — for end-to-end auth, sign-then-encrypt (nested JWT). See [[JWT FN 02 - Anatomy and Encoding]].

💡 **Takeaway:** Reach for joserfc when you need JWE, full JWK Set management, or you're escaping `python-jose`. For plain JWT signing/verification, [[JWT PR 01 - PyJWT]] is lighter.
