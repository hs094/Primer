# AL 04 — EdDSA (Ed25519, Ed448)

🔑 **One-line: modern, deterministic, fast asymmetric signatures over Edwards curves — added to JOSE by RFC 8037; the recommended default for new JWT systems.**

## How it works

RFC 8037 ("CFRG Elliptic Curve Diffie-Hellman (ECDH) and Signatures in JOSE") adds EdDSA from RFC 8032 to JOSE. There is **one** `alg` value — `EdDSA` — and the curve is selected by the **key**, not the algorithm string.

> *"The parameter 'alg' value is set to 'EdDSA'."* — RFC 8037

> *"The EdDSA variant used is determined by the subtype of the key (Ed25519 for 'Ed25519' and Ed448 for 'Ed448')."* — RFC 8037

| Curve | Public key | Signature | Approx security |
|---|---|---|---|
| Ed25519 | 32 bytes | 64 bytes | ~128-bit |
| Ed448 | 57 bytes | 114 bytes | ~224-bit |

Three properties make EdDSA attractive vs ECDSA:

1. **Deterministic.** *"Signing and non-batch signature verification are deterministic operations and do not need random numbers of any kind."* — RFC 8037. No `k` to leak, no PRNG to fail.
2. **Rigid curves.** Curve25519 / Curve448 parameters are explained — none of the unexplained NIST seeds.
3. **Fast.** Ed25519 signs/verifies faster than ECDSA P-256 in most libraries; tiny 32-byte public keys.

## In a JWT header

```json
{ "alg": "EdDSA", "typ": "JWT", "kid": "ed25519-2026-q2" }
```

The header carries `EdDSA` regardless of curve. The verifier looks up the key by `kid` and infers Ed25519 vs Ed448 from the JWK's `crv`.

### Matching JWK shape

EdDSA keys use `kty: "OKP"` (Octet Key Pair) — the JOSE key type added by RFC 8037 specifically for these curves:

```json
{
  "kty": "OKP",
  "crv": "Ed25519",
  "kid": "ed25519-2026-q2",
  "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo"
}
```

Private keys add a `d` field (base64url, 32 or 57 raw bytes). See [[JWT JO 05 - JWK]].

> *"These subtypes MUST NOT be used for Elliptic Curve Diffie-Hellman Ephemeral Static (ECDH-ES)."* — RFC 8037

(Ed25519 / Ed448 are signature-only; X25519 / X448 are the ECDH variants.)

## Signing with PyJWT

```python
import jwt
from pathlib import Path

private_key = Path("ed25519_private.pem").read_bytes()
public_key = Path("ed25519_public.pem").read_bytes()

token = jwt.encode(
    {"sub": "user-123"},
    private_key,
    algorithm="EdDSA",
    headers={"kid": "ed25519-2026-q2"},
)

claims = jwt.decode(token, public_key, algorithms=["EdDSA"])
```

Requires `pyjwt[crypto]` (which pulls in `cryptography` ≥ 2.6 for Ed25519, ≥ 2.6 for Ed448). Install with `uv add 'pyjwt[crypto]'`.

## Key requirements

Curve choice picks the strength — no extra knobs:

- **Ed25519** for ~128-bit security. The default for almost every greenfield project.
- **Ed448** for ~224-bit security and post-quantum margin paranoia. Slower; not as widely supported.

Generate an Ed25519 keypair:

```bash
# Private key (PKCS#8 PEM)
openssl genpkey -algorithm ed25519 -out ed25519_private.pem

# Public key
openssl pkey -in ed25519_private.pem -pubout -out ed25519_public.pem
```

For Ed448:

```bash
openssl genpkey -algorithm ed448 -out ed448_private.pem
openssl pkey -in ed448_private.pem -pubout -out ed448_public.pem
```

⚠️ **Library / runtime support is uneven.** Ed25519 JWT support landed late: Node `jsonwebtoken` ≥ 9.0, Python `PyJWT` ≥ 2.0 with `cryptography` ≥ 2.6, Go `golang-jwt/jwt/v5`. If you have legacy verifiers stuck on older versions, stick with [[JWT AL 02 - RSA Signatures]] or [[JWT AL 03 - ECDSA]] until they upgrade.

⚠️ **Single `alg` value, multiple curves.** Because both Ed25519 and Ed448 share `alg: EdDSA`, the verifier *must* check the key's curve matches what's expected for the `kid`. Don't trust `alg` to disambiguate.

⚠️ **Algorithm confusion (EdDSA→HS).** Same family of attack as RS→HS: pinning `algorithms=` is mandatory. See [[JWT SE 02 - Algorithm Confusion]] and [[JWT SE 01 - RFC 8725 BCP]].

⚠️ **Key-binding caveat.** Per RFC 8037: *"Although for Ed25519 and Ed448, the signature binds the key used for signing, do not assume this, as there are many signature algorithms that fail to make such a binding."* Don't rely on signature → key binding for anything security-relevant; bind via the issuer's identity / `kid` lookup.

⚠️ **No `k` to mess up — but key generation still matters.** EdDSA dodges the ECDSA `k` footgun, but the long-term private key still needs a strong RNG at generation time. Generate keys on the host that will use them; don't generate on a low-entropy device.

⚠️ **Not all HSMs / KMS support Ed25519 yet.** AWS KMS gained Ed25519 in 2023; some older HSMs still don't. Check before architecting around hardware-backed signing.

## When to pick EdDSA

- Greenfield system, no legacy verifier constraints.
- You want the smallest keys/signatures + fastest signing without the ECDSA `k` hazard.
- Modern stack: PyJWT ≥ 2, `jsonwebtoken` ≥ 9, Go ≥ 1.13, `cryptography` ≥ 2.6.

**Stick with RS256 when** you must interop with OIDC providers that don't expose EdDSA keys (most public IdPs as of 2026 still default to RS256).

## Related

- [[JWT FN 01 - What Is a JWT]]
- [[JWT FN 02 - Anatomy and Encoding]]
- [[JWT JO 02 - JWS]]
- [[JWT JO 05 - JWK]]
- [[JWT JO 06 - JWA]]
- [[JWT PR 01 - PyJWT]]
- [[JWT PR 03 - jsonwebtoken Node]]
- [[JWT AL 01 - HMAC]]
- [[JWT AL 02 - RSA Signatures]]
- [[JWT AL 03 - ECDSA]]
- [[JWT AL 05 - none Algorithm]]

💡 **Takeaway:** EdDSA (Ed25519) is the "what should I pick today" answer — deterministic, tiny, fast, and free of ECDSA's nonce hazard, as long as your verifiers support it.
