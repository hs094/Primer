# AL 01 — HMAC (HS256, HS384, HS512)

🔑 **One-line: symmetric MAC — same secret signs and verifies; the simplest and fastest JWT signature, but only safe when both sides are trusted (no public verifiers).**

## How it works

HMAC ("Hash-based Message Authentication Code") combines a shared secret key with a cryptographic hash function per [RFC 2104](https://datatracker.ietf.org/doc/html/rfc2104). RFC 7518 §3.2 defines three JOSE variants over SHA-2:

| `alg` | Hash | MAC size | Min key size |
|---|---|---|---|
| `HS256` | SHA-256 | 32 bytes | 256 bits |
| `HS384` | SHA-384 | 48 bytes | 384 bits |
| `HS512` | SHA-512 | 64 bytes | 512 bits |

The MAC is computed over the JWS Signing Input — `BASE64URL(header) || '.' || BASE64URL(payload)` — using the shared key. Verification recomputes the MAC and compares; per RFC 7518: *"The comparison of the computed HMAC value to the JWS Signature value MUST be done in a constant-time manner to thwart timing attacks."*

Because the **same key signs and verifies**, HMAC is unsuitable for any flow where a third party needs to verify tokens without being trusted to mint them. For that, see [[JWT AL 02 - RSA Signatures]] or [[JWT AL 03 - ECDSA]].

## In a JWT header

```json
{ "alg": "HS256", "typ": "JWT" }
```

## Signing with PyJWT

```python
import jwt

key = "secret"
encoded = jwt.encode({"some": "payload"}, key, algorithm="HS256")
decoded = jwt.decode(encoded, key, algorithms=["HS256"])
# {'some': 'payload'}
```

Always pass `algorithms=` as a **list of explicitly allowed algorithms** to `jwt.decode` — never let the token's own `alg` header decide. See [[JWT SE 02 - Algorithm Confusion]].

### Enforcing minimum key length

PyJWT only warns by default on short keys. To make it raise:

```python
import jwt

strict_jwt = jwt.PyJWT(options={"enforce_minimum_key_length": True})
try:
    strict_jwt.encode({"some": "payload"}, "short", algorithm="HS256")
except jwt.InvalidKeyError:
    print("key too short")
```

Per PyJWT docs: *"Key must be at least as long as the hash output (32, 48, or 64 bytes respectively)."*

### Custom headers (e.g. `kid` for key rotation)

```python
jwt.encode(
    {"some": "payload"},
    "secret",
    algorithm="HS256",
    headers={"kid": "230498151c214b788dd97f22b85410a5"},
)
```

The `kid` lets verifiers look up the right secret when you rotate keys. See [[JWT JO 05 - JWK]] and [[JWT SE 03 - Key Management]].

## Key requirements

RFC 7518 §3.2: *"A key of the same size as the hash output (for instance, 256 bits for `HS256`) or larger MUST be used with this algorithm."*

| `alg` | Min bytes | Min bits |
|---|---|---|
| `HS256` | 32 | 256 |
| `HS384` | 48 | 384 |
| `HS512` | 64 | 512 |

Generate a key with proper entropy:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
# 32 bytes = 256 bits, suitable for HS256
```

Or:

```bash
openssl rand -base64 32
```

⚠️ **Never use a passphrase, dictionary word, or `"secret"` as your HMAC key.** Anything with low entropy is offline-brute-forceable: an attacker who captures one token can grind candidate keys until the recomputed MAC matches, then forge arbitrary tokens. Tools like `hashcat -m 16500` exist specifically for this.

⚠️ **Algorithm-confusion / RS→HS swap.** If your verifier accepts both `RS256` and `HS256` (or naively trusts the token's `alg`), an attacker can take your **public** RSA key, swap `alg` to `HS256`, and sign a forged token using the public key bytes as the HMAC secret. Verifier recomputes HMAC with the same public key and accepts it. Mitigation: always pin `algorithms=["HS256"]` (or `["RS256"]`) at the call site. See [[JWT SE 02 - Algorithm Confusion]] and [[JWT SE 01 - RFC 8725 BCP]].

⚠️ **Symmetric = no public verification.** If you ship the secret to clients, any client can mint tokens. HMAC is for server-to-server or single-tenant systems only.

⚠️ **Timing attacks on hand-rolled compare.** RFC 7518 mandates constant-time comparison. PyJWT does this for you; rolling your own with `==` on the bytes leaks the MAC byte-by-byte.

⚠️ **Key reuse across services.** One leaked secret compromises every service that shares it. Rotate per-service, version with `kid`. See [[JWT SE 03 - Key Management]].

## When to pick HMAC

- Internal service-to-service auth where both sides hold the secret.
- Short-lived session tokens minted and verified by the same monolith.
- You want minimal CPU cost and dead-simple operations.

**Don't pick HMAC when** verifiers should not be able to mint — use [[JWT AL 02 - RSA Signatures]], [[JWT AL 03 - ECDSA]], or [[JWT AL 04 - EdDSA]].

## Related

- [[JWT FN 01 - What Is a JWT]]
- [[JWT FN 02 - Anatomy and Encoding]]
- [[JWT JO 02 - JWS]]
- [[JWT JO 06 - JWA]]
- [[JWT PR 01 - PyJWT]]
- [[JWT PR 03 - jsonwebtoken Node]]
- [[JWT AL 02 - RSA Signatures]]
- [[JWT AL 03 - ECDSA]]
- [[JWT AL 04 - EdDSA]]
- [[JWT AL 05 - none Algorithm]]

💡 **Takeaway:** HS256 is fine — *if* the secret is 32+ random bytes, only servers hold it, and you pin `algorithms=["HS256"]` on every verify call.
