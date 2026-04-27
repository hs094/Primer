# JO 06 — JWA (JSON Web Algorithms)

🔑 **JWA is the algorithm registry: it defines exactly which strings (`HS256`, `RS256`, `ES256`, `RSA-OAEP-256`, `A256GCM`, `dir`, `none`, …) mean what and which are MUST/SHOULD/MAY for a conforming implementation.**

## What JWA is (RFC 7518)

JWA backs three other specs:

- **JWS `alg`** → digital sig / MAC algorithm
- **JWE `alg`** → key management algorithm
- **JWE `enc`** → content encryption (AEAD) algorithm
- Plus `kty` / `crv` mappings used inside [[JWT JO 05 - JWK]]

Implementation-requirement levels: **Required**, **Recommended+ / Recommended / Recommended-**, **Optional**. Levels can shift over time as crypto erodes.

## Signature / MAC algorithms (used in [[JWT JO 02 - JWS]])

| `alg` | Family | Key | RFC level | Notes |
|---|---|---|---|---|
| `HS256` | HMAC-SHA-256 | `oct`, ≥ 256 bits | **Required** | See [[JWT AL 01 - HMAC]] |
| `HS384` | HMAC-SHA-384 | `oct`, ≥ 384 bits | Optional | |
| `HS512` | HMAC-SHA-512 | `oct`, ≥ 512 bits | Optional | |
| `RS256` | RSASSA-PKCS1-v1_5 + SHA-256 | RSA ≥ 2048 | **Recommended** | See [[JWT AL 02 - RSA Signatures]] |
| `RS384` / `RS512` | + SHA-384/512 | RSA ≥ 2048 | Optional | |
| `PS256` | RSASSA-PSS + MGF1-SHA-256 | RSA ≥ 2048 | Optional | Preferred over RS* |
| `PS384` / `PS512` | + SHA-384/512 | RSA ≥ 2048 | Optional | |
| `ES256` | ECDSA on **P-256** + SHA-256 | EC P-256 | **Recommended+** | See [[JWT AL 03 - ECDSA]] |
| `ES384` | ECDSA on **P-384** + SHA-384 | EC P-384 | Optional | |
| `ES512` | ECDSA on **P-521** + SHA-512 | EC P-521 | Optional | (yes, 521-bit curve, 512-bit hash) |
| `EdDSA` | Ed25519 / Ed448 (RFC 8037) | OKP | Optional | See [[JWT AL 04 - EdDSA]] |
| `none` | **No signature** | — | Optional | See [[JWT AL 05 - none Algorithm]] |

> "Implementations that support Unsecured JWSs MUST NOT accept such objects as valid unless the application specifies that it is acceptable." — RFC 7518 §3.6

## Key Management algorithms (used in [[JWT JO 04 - JWE]] `alg`)

| `alg` | What it does | Class | RFC level |
|---|---|---|---|
| `RSA1_5` | RSAES-PKCS1-v1_5 wrap of CEK | Key Encryption | Recommended- (Bleichenbacher) |
| `RSA-OAEP` | RSA-OAEP w/ SHA-1 + MGF1 | Key Encryption | Recommended+ |
| `RSA-OAEP-256` | RSA-OAEP w/ SHA-256 + MGF1 | Key Encryption | Optional (preferred today) |
| `A128KW` | AES-128 Key Wrap (RFC 3394) | Key Wrapping | Recommended |
| `A192KW` | AES-192 Key Wrap | Key Wrapping | Optional |
| `A256KW` | AES-256 Key Wrap | Key Wrapping | Recommended |
| `dir` | Pre-shared symmetric key IS the CEK | Direct Encryption | Recommended |
| `ECDH-ES` | Ephemeral-Static ECDH → CEK direct | Direct Key Agreement | Recommended+ |
| `ECDH-ES+A128KW` | ECDH-ES → KEK → AES-128-KW | Key Agreement w/ Key Wrap | Recommended |
| `ECDH-ES+A192KW` | …A192-KW | same | Optional |
| `ECDH-ES+A256KW` | …A256-KW | same | Recommended |
| `A128GCMKW` | AES-128-GCM key wrap | Key Wrapping w/ AEAD | Optional |
| `A192GCMKW` | AES-192-GCM key wrap | same | Optional |
| `A256GCMKW` | AES-256-GCM key wrap | same | Optional |
| `PBES2-HS256+A128KW` | Password-derived key + A128KW | Key Wrapping | Optional |
| `PBES2-HS384+A192KW` | …+A192KW | same | Optional |
| `PBES2-HS512+A256KW` | …+A256KW | same | Optional |

## Content Encryption algorithms (JWE `enc`, all AEAD)

| `enc` | Construction | Key bits | RFC level |
|---|---|---|---|
| `A128CBC-HS256` | AES-128-CBC + HMAC-SHA-256 (encrypt-then-MAC) | 256 (split 128+128) | **Required** |
| `A192CBC-HS384` | AES-192-CBC + HMAC-SHA-384 | 384 | Optional |
| `A256CBC-HS512` | AES-256-CBC + HMAC-SHA-512 | 512 | **Required** |
| `A128GCM` | AES-128-GCM | 128 | Optional |
| `A192GCM` | AES-192-GCM | 192 | Optional |
| `A256GCM` | AES-256-GCM | 256 | **Recommended** |

`A*CBC-HS*` is the JWE-specific construction, **not** raw AES-CBC — the spec mandates encrypt-then-MAC with a split key, and verifiers MUST check the tag in constant time.

## `kty` / `crv` parameter mappings

JWA also defines the per-`kty` JWK members ([[JWT JO 05 - JWK]] uses these):

| `kty` | Required public | Private | `crv` values |
|---|---|---|---|
| `RSA` | `n`, `e` | `d`, `p`, `q`, `dp`, `dq`, `qi` | — |
| `EC` | `crv`, `x`, `y` | `d` | `P-256`, `P-384`, `P-521` |
| `oct` | — | `k` | — |
| `OKP` (RFC 8037, not in 7518) | `crv`, `x` | `d` | `Ed25519`, `Ed448`, `X25519`, `X448` |

## Picking algorithms today (2026 baseline)

- **JWS signing (asymmetric):** `EdDSA` (Ed25519) or `ES256`. Falls back to `PS256` if you need RSA. Avoid `RS*` for new designs.
- **JWS signing (symmetric):** `HS256` with a 256-bit random key, only when both ends are first-party services.
- **JWE key management:** `ECDH-ES+A256KW` for asymmetric recipients, `dir` only for short-lived first-party flows. Avoid `RSA1_5`.
- **JWE content:** `A256GCM`. Use `A256CBC-HS512` only if a peer can't do GCM.

⚠️ **Footguns:**
- **`none` exists.** A library that lets the token's `alg` field choose the verifier function, including `none`, is broken. Always allow-list. See [[JWT AL 05 - none Algorithm]] and [[JWT SE 02 - Algorithm Confusion]].
- **`HS256` ↔ `RS256` confusion** — if your verifier accepts both and resolves the key generically, an attacker swaps `alg` to `HS256` and HMACs with the public RSA key as the secret.
- **`RSA1_5` is Bleichenbacher-vulnerable** in many implementations. Treat as deprecated.
- **`RSA-OAEP` (the SHA-1 variant) is not equivalent to `RSA-OAEP-256`** — pick the latter unless interop forces otherwise.
- **`ES512` uses curve P-521 + SHA-512.** The `512` is the *hash*, not the curve. Easy to miswire.
- **`A*CBC-HS*` requires the **encrypt-then-MAC** construction defined in §5.2.** Off-the-shelf AES-CBC + HMAC code is usually *not* compatible.
- **PBES2 uses a password — slow on purpose.** `p2c` (iteration count) must be high; verifying many tokens this way is a DoS surface.
- **Implementation level can change.** RFC 7518 explicitly says levels may shift; treat the table as a starting point and consult current security guidance (RFC 8725 BCP).

See:
- [[JWT JO 02 - JWS]] for how `alg` fits the signing flow
- [[JWT JO 04 - JWE]] for `alg` (key mgmt) vs `enc` (content) split
- [[JWT JO 05 - JWK]] for how `kty`/`crv` parameters land on the JWK side

💡 **Takeaway:** JWA is a registry of strings. Pick a small allow-list per surface (sign verify / key mgmt / content enc), pin it server-side, and never let the token decide what's acceptable.
