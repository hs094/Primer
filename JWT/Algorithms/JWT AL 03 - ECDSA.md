# AL 03 — ECDSA (ES256, ES384, ES512)

🔑 **One-line: asymmetric signatures over NIST elliptic curves — same trust model as RSA but with much smaller keys and signatures (~64 bytes for ES256 vs ~256 for RS256).**

## How it works

RFC 7518 §3.4 defines three ECDSA variants, each pinning a curve to a hash:

| `alg` | Curve | Hash | Signature size |
|---|---|---|---|
| `ES256` | P-256 (secp256r1) | SHA-256 | 64 bytes (R 32 ∥ S 32) |
| `ES384` | P-384 (secp384r1) | SHA-384 | 96 bytes (R 48 ∥ S 48) |
| `ES512` | **P-521** (secp521r1) | SHA-512 | 132 bytes (R 66 ∥ S 66) |

Note the off-by-one: `ES512` uses **P-521**, not P-512. The 521-bit field elements are zero-padded to 66 octets each.

Per RFC 7518 §3.4, the JWS signature is the **fixed-length R ∥ S concatenation in big-endian**, *not* the ASN.1 DER `SEQUENCE { r INTEGER, s INTEGER }` that OpenSSL emits natively. This is a common interop trap — see ⚠️ below.

> *"…the JWS Signature value [is] the concatenation of the (R, S) pair octet sequences in that order. The result is a 64-octet sequence."* — RFC 7518 §3.4 (for ES256)

## In a JWT header

```json
{ "alg": "ES256", "typ": "JWT", "kid": "ec-key-1" }
```

## Signing with PyJWT

```python
import jwt
from pathlib import Path

private_key = Path("ec_private.pem").read_bytes()
public_key = Path("ec_public.pem").read_bytes()

token = jwt.encode(
    {"sub": "user-123"},
    private_key,
    algorithm="ES256",
    headers={"kid": "ec-key-1"},
)

claims = jwt.decode(token, public_key, algorithms=["ES256"])
```

PyJWT requires the `cryptography` extra: `uv add 'pyjwt[crypto]'`. The library handles the ASN.1↔raw R∥S conversion for you.

## Key requirements

Curve choice **is** the key strength — there's no "key size" knob beyond curve selection.

| Curve | Approx security |
|---|---|
| P-256 | ~128-bit |
| P-384 | ~192-bit |
| P-521 | ~256-bit |

For most applications, **`ES256` is the right default** — comparable security to RS256, much smaller tokens, and faster signing.

Generate a P-256 keypair:

```bash
# P-256 private key
openssl ecparam -name prime256v1 -genkey -noout -out ec_private.pem

# Public key
openssl ec -in ec_private.pem -pubout -out ec_public.pem
```

For P-384 use `-name secp384r1`; for P-521 use `-name secp521r1`.

⚠️ **Nonce reuse = total private-key recovery.** ECDSA signatures use a per-signature random `k`. If `k` is ever reused (or predictable) across two signatures with the same key, the private key can be recovered with elementary algebra — the [Sony PS3 hack (2010)](https://fahrplan.events.ccc.de/congress/2010/Fahrplan/events/4087.en.html) was exactly this. Modern libraries (PyJWT via `cryptography`, Go `crypto/ecdsa` ≥1.19) use [RFC 6979 deterministic-k](https://datatracker.ietf.org/doc/html/rfc6979) to eliminate this. **Don't roll your own ECDSA.** Prefer [[JWT AL 04 - EdDSA]] which sidesteps this entirely.

⚠️ **Signature format mismatch.** RFC 7518 mandates raw fixed-length R∥S. OpenSSL's CLI and many low-level APIs produce ASN.1 DER (`30 44 02 20 ...`). If you hand-craft signatures or interop with non-JOSE tooling, you must convert. PyJWT and `node-jose`/`jsonwebtoken` do this correctly out of the box.

⚠️ **`ES512` uses P-521, not P-512.** Verifiers that allocate 64-byte buffers for "ES512" R/S will silently truncate. Always allocate based on the curve, not the algorithm string.

⚠️ **Curve confusion.** Don't accept `ES256` *and* `ES384` *and* `ES512` on the same endpoint without `kid`-bound key/curve mapping. Mismatched curve and hash combinations are not interoperable and can mask attacks. Pin `algorithms=["ES256"]`. See [[JWT SE 02 - Algorithm Confusion]].

⚠️ **NIST curve trust.** P-256/384/521 are NIST/SECG curves with unexplained seed parameters that have drawn cryptographer skepticism. If that concerns your threat model, use [[JWT AL 04 - EdDSA]] (Ed25519) which uses the rigid Curve25519 parameters.

⚠️ **Algorithm confusion (EC→HS).** Same shape as the RS→HS attack: an attacker takes your **EC public key bytes** and tries to use them as an HMAC secret with `alg: HS256`. Pin `algorithms=` explicitly. See [[JWT SE 02 - Algorithm Confusion]] and [[JWT SE 01 - RFC 8725 BCP]].

## When to pick ECDSA

- Same trust model as RSA (public verifiers, private signers) but you want smaller tokens.
- Mobile/IoT clients where every header byte matters.
- Existing infra already has NIST-curve EC keys (e.g. TLS certs, HSMs).

**Prefer EdDSA when** starting greenfield — it's faster, smaller, deterministic, and avoids the `k` footgun. See [[JWT AL 04 - EdDSA]].

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
- [[JWT AL 04 - EdDSA]]
- [[JWT AL 05 - none Algorithm]]

💡 **Takeaway:** ES256 gives RSA-equivalent security with 4× smaller signatures — pick it when token size or signing speed matters and your library uses RFC 6979 deterministic-k.
