# JO 05 — JWK (JSON Web Key)

🔑 **A JWK is a key as a JSON object; a JWK Set is `{"keys":[...]}`. `kty` decides which extra members are required.**

## What JWK is (RFC 7517)

> "A JSON Web Key (JWK) is a JavaScript Object Notation (JSON) data structure that represents a cryptographic key." — RFC 7517 §1

> "The member names within a JWK MUST be unique." — §4

JWK is the canonical way to publish, discover, and rotate keys for [[JWT JO 02 - JWS]] and [[JWT JO 04 - JWE]]. OIDC uses a JWK Set at `/.well-known/jwks.json` to publish issuer signing keys.

## Common parameters (apply to every `kty`)

| Param | Meaning | Notes |
|---|---|---|
| `kty` | **Key Type** | **REQUIRED.** `RSA`, `EC`, `oct`, `OKP` (RFC 8037 — Ed25519/X25519) |
| `use` | Public-key use | `sig` or `enc`. Mutually exclusive with `key_ops` |
| `key_ops` | Permitted ops (array) | `sign`, `verify`, `encrypt`, `decrypt`, `wrapKey`, `unwrapKey`, `deriveKey`, `deriveBits` |
| `alg` | Intended algorithm | e.g. `RS256`, `ES256`, `RSA-OAEP-256`, `A128KW` |
| `kid` | Key ID | Free-form string. Used to match the `kid` in a JOSE header |
| `x5u` | X.509 URL | Cert chain (PEM) for the same key. TLS required |
| `x5c` | X.509 cert chain | JSON array of standard-base64 (NOT base64url) DER certs |
| `x5t` | SHA-1 cert thumbprint | Deprecated for new use |
| `x5t#S256` | SHA-256 cert thumbprint | Preferred |

## `kty`-specific members

| `kty` | Public params | Private params | Curves / sizes |
|---|---|---|---|
| `RSA` | `n` (modulus), `e` (exponent) | `d`, `p`, `q`, `dp`, `dq`, `qi`, optional `oth` | ≥ 2048-bit modulus |
| `EC` | `crv`, `x`, `y` | `d` | `P-256`, `P-384`, `P-521` |
| `OKP` (RFC 8037) | `crv`, `x` | `d` | `Ed25519`, `Ed448`, `X25519`, `X448` |
| `oct` | — | `k` (the raw symmetric key, base64url) | length-defined by `alg` |

All numeric values (`n`, `e`, `d`, `x`, `y`, …) are **base64url** big-endian unsigned integers — never decimal, never base64.

## Worked examples

**RSA public key** (signing — issuer JWKS entry):

```json
{
  "kty": "RSA",
  "use": "sig",
  "alg": "RS256",
  "kid": "2026-04-rotation",
  "n":   "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECP...",
  "e":   "AQAB"
}
```

**EC public key** (encryption — ECDH-ES recipient):

```json
{
  "kty": "EC",
  "use": "enc",
  "crv": "P-256",
  "x":   "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
  "y":   "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM",
  "kid":  "1"
}
```

**Symmetric key** (HMAC / AES-KW):

```json
{
  "kty": "oct",
  "alg": "A128KW",
  "k":   "GawgguFyGrWKav7AX4VKUg"
}
```

**Ed25519 (`OKP`, RFC 8037)**:

```json
{
  "kty": "OKP",
  "crv": "Ed25519",
  "x":   "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
  "kid": "ed-2026-04"
}
```

## JWK Set (§5)

A JWK Set is a JSON object with a **required** `keys` array of JWKs. Unknown members MUST be ignored.

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "2026-03-rotation",
      "alg": "RS256",
      "n": "...", "e": "AQAB"
    },
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "2026-04-rotation",
      "alg": "RS256",
      "n": "...", "e": "AQAB"
    }
  ]
}
```

Common location: `https://issuer.example.com/.well-known/jwks.json` (OIDC Discovery `jwks_uri`).

Verifier flow:
1. Pull JWS header → read `kid` and `alg`.
2. Fetch + cache the JWK Set from the trusted `jwks_uri` (NOT from `jku` in the token).
3. Pick the JWK with matching `kid` (and `use=sig`, and `alg` consistent with the JOSE header).
4. Reconstruct the key and verify per [[JWT JO 02 - JWS]].

## Operational concerns

- **Rotation:** publish new key under a new `kid` *before* signing with it. Verifiers cache the JWKS; give them a window. Don't drop old keys until all in-flight tokens have expired.
- **Caching:** respect HTTP cache headers; refresh on `kid` miss; have a max-refresh-rate to prevent JWKS-flood DoS.
- **Public vs private exposure:** an RSA public JWK has `n`, `e` only. The presence of `d` (or `p`/`q`/`k`/private `d` for EC) means it's a **private** key. Never publish those.

⚠️ **Footguns:**
- **`x5c` is *standard* base64**, not base64url. Don't reuse the JWS decoder.
- **`use` and `key_ops` together is forbidden** if they conflict; pick one.
- **`alg` is informational, not enforcement.** A JWS verifier still has to allow-list algorithms — a JWK saying `alg: RS256` doesn't stop someone presenting an `HS256` token.
- **Trusting `jku`/`jwk` from the JOSE header is the canonical key-confusion attack** — the attacker hosts their own JWK Set. See [[JWT SE 02 - Algorithm Confusion]] and [[JWT SE 03 - Key Management]]. Always pin `jwks_uri` from issuer metadata.
- **`kid` collisions across rotations.** Make `kid` globally unique per issuer; never reuse.
- **Symmetric `oct` keys must never leave the boundary** — publishing them in a JWK Set is by definition leaking the secret.

See [[JWT JO 06 - JWA]] for which `kty` + `crv` combos pair with which algorithms.

💡 **Takeaway:** JWK = `{kty, ...kty-specific...}` as JSON, with optional metadata (`kid`, `use`, `alg`). JWK Set = `{"keys":[...]}` published at `jwks_uri`. Match by `kid`, validate the `kty`/`alg` combo, and never trust JWKS pointers that came from the token.
