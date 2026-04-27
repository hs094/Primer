# AL 02 — RSA Signatures (RS256/384/512, PS256/384/512)

🔑 **One-line: asymmetric RSA signatures — private key signs, public key verifies; ubiquitous for OIDC/OAuth because verifiers (clients, APIs) only ever see the public key.**

## How it works

RFC 7518 defines two RSA signature families, both producing a signature over the JWS Signing Input.

### §3.3 — RSASSA-PKCS1-v1_5 (RS256/384/512)

Classic RFC 8017 PKCS#1 v1.5 padding plus a SHA-2 hash. Deterministic — the same key + message always produces the same signature.

| `alg` | Hash | Status |
|---|---|---|
| `RS256` | SHA-256 | Recommended |
| `RS384` | SHA-384 | Optional |
| `RS512` | SHA-512 | Optional |

> *"generate a digital signature of the JWS Signing Input using RSASSA-PKCS1-v1_5-SIGN and the SHA-256 hash function with the desired private key."* — RFC 7518 §3.3

### §3.5 — RSASSA-PSS (PS256/384/512)

Probabilistic Signature Scheme from RFC 8017. Uses **MGF1** as the mask generation function with the **same SHA-2** for hash and MGF, plus a random salt. Two PSS signatures of the same payload differ.

| `alg` | Hash + MGF1 | Salt size |
|---|---|---|
| `PS256` | SHA-256 | 32 bytes |
| `PS384` | SHA-384 | 48 bytes |
| `PS512` | SHA-512 | 64 bytes |

> *"The size of the salt value is the same size as the hash function output."* — RFC 7518 §3.5

PSS has a tighter security reduction to the RSA assumption than PKCS1-v1_5 and is preferred for new designs — but RS256 dominates in the wild because every OIDC provider implements it.

## In a JWT header

```json
{ "alg": "RS256", "typ": "JWT", "kid": "key-2026-q2" }
```

For PSS:

```json
{ "alg": "PS256", "typ": "JWT", "kid": "key-2026-q2" }
```

## Signing with PyJWT

```python
import jwt
from pathlib import Path

private_key = Path("private.pem").read_bytes()
public_key = Path("public.pem").read_bytes()

token = jwt.encode(
    {"sub": "user-123"},
    private_key,
    algorithm="RS256",
    headers={"kid": "key-2026-q2"},
)

claims = jwt.decode(token, public_key, algorithms=["RS256"])
```

Swap `RS256` for `PS256` to use PSS — PyJWT supports both via the `cryptography` backend (install with `pip install pyjwt[crypto]`, or `uv add 'pyjwt[crypto]'`).

PyJWT accepts PEM-encoded keys (`-----BEGIN PRIVATE KEY-----` / `-----BEGIN PUBLIC KEY-----`) directly as `bytes` or `str`, or pre-loaded `cryptography` key objects.

## Key requirements

Both §3.3 and §3.5: *"A key of size 2048 bits or larger MUST be used with these algorithms."*

- **2048-bit minimum.** Below that is non-compliant and breakable on a long enough horizon.
- **3072-bit** is the NIST recommendation through 2030 for new systems.
- **4096-bit** for long-lived signing keys; verification cost rises but is still cheap.

Generate a keypair:

```bash
# 2048-bit RSA private key (PKCS#8 PEM)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem

# Extract the public key
openssl rsa -in private.pem -pubout -out public.pem
```

For larger keys, change `2048` to `3072` or `4096`.

⚠️ **`alg: none` substitution.** If your verifier accepts `none` (or has `algorithms` unset), the attacker strips the signature entirely. See [[JWT AL 05 - none Algorithm]].

⚠️ **RS→HS algorithm confusion.** This is *the* JWT footgun. Attacker takes your published public key, sets `alg: HS256`, and signs a forged token using the **public key's PEM bytes as the HMAC secret**. A naive verifier that auto-selects the algorithm from the header HMACs the token with the same public key bytes and accepts. **Always pin `algorithms=["RS256"]` (or `["PS256"]`) explicitly.** See [[JWT SE 02 - Algorithm Confusion]] and [[JWT SE 01 - RFC 8725 BCP]].

⚠️ **Don't share the private key across environments.** Staging and prod should have separate keypairs. A leaked staging key with broad `aud` lets an attacker mint prod-valid tokens.

⚠️ **PKCS1-v1_5 vs PSS — pick one and stick.** PSS is theoretically stronger but ecosystem support is thinner. RS256 is the universal default for OIDC ID tokens. Don't accept *both* on a single endpoint without strict `kid`-based key separation.

⚠️ **Signature size.** RSA signatures are large: 256 bytes for 2048-bit, 512 for 4096-bit. That bloats every header (`Authorization: Bearer ...`) — consider [[JWT AL 03 - ECDSA]] or [[JWT AL 04 - EdDSA]] when token size matters.

⚠️ **Key rotation requires `kid`.** Without a `kid` header, you can't run two valid keys in parallel during rotation. Publish all current keys via a JWKS endpoint and require `kid` for lookup. See [[JWT JO 05 - JWK]] and [[JWT SE 03 - Key Management]].

## When to pick RSA

- Public OIDC/OAuth flows where many parties verify but only one issues.
- Federated systems with a JWKS endpoint.
- Long-lived signing keys (rotation is rare; verifiers cache JWKS for hours).

**Prefer ECDSA or EdDSA when** you care about signature size, signing speed, or modern crypto hygiene.

## Related

- [[JWT FN 01 - What Is a JWT]]
- [[JWT FN 02 - Anatomy and Encoding]]
- [[JWT JO 02 - JWS]]
- [[JWT JO 05 - JWK]]
- [[JWT JO 06 - JWA]]
- [[JWT PR 01 - PyJWT]]
- [[JWT PR 03 - jsonwebtoken Node]]
- [[JWT AL 01 - HMAC]]
- [[JWT AL 03 - ECDSA]]
- [[JWT AL 04 - EdDSA]]
- [[JWT AL 05 - none Algorithm]]

💡 **Takeaway:** RS256 with a 2048+ bit key, `kid` headers, and pinned `algorithms=["RS256"]` is the safe default for any system where verifiers shouldn't be able to mint.
