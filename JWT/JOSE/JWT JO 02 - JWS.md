# JO 02 — JWS (JSON Web Signature)

🔑 **A JWS binds a payload to a signature over `BASE64URL(header) || '.' || BASE64URL(payload)` — the signing input never includes the signature itself.**

## What JWS is (RFC 7515)

> "JSON Web Signature (JWS) represents content secured with digital signatures or Message Authentication Codes (MACs) using JSON-based data structures." — RFC 7515 §1

Three logical parts:

1. **JOSE Header** — JSON describing the signing operation (alg, kid, …)
2. **JWS Payload** — the bytes being secured (arbitrary; for a JWT, a claims JSON)
3. **JWS Signature** — digital sig or MAC over the **Signing Input**

The **Signing Input** is exactly:

```
ASCII(BASE64URL(UTF8(JWS Protected Header)) || '.' || BASE64URL(JWS Payload))
```

Note the signature is computed over the **base64url-encoded** header and payload, not the raw JSON. This makes the operation byte-stable across JSON whitespace differences.

## Header parameter catalog (§4)

| Param | Meaning | Notes |
|---|---|---|
| `alg` | **Algorithm** identifying the sig/MAC | **REQUIRED.** Must be understood. e.g. `HS256`, `RS256`, `ES256`, `EdDSA`, `none`. See [[JWT JO 06 - JWA]] |
| `jku` | **JWK Set URL** — URI to fetch a public-key set | MUST use TLS, MUST verify server identity |
| `jwk` | **JSON Web Key** — public key inlined | Public keys only (never private) |
| `kid` | **Key ID** — hint to select a key | Case-sensitive string. Matches JWK `kid` |
| `x5u` | **X.509 URL** — URI to a PEM cert chain | TLS required; first cert is the signer |
| `x5c` | **X.509 Cert Chain** — array of base64 (NOT base64url) DER certs | Verifier MUST validate chain per RFC 5280 |
| `x5t` | **SHA-1 thumbprint** of the signer cert (DER) | Deprecated for new use — collisions |
| `x5t#S256` | **SHA-256 thumbprint** of the signer cert | Preferred over `x5t` |
| `typ` | **Type** of the whole object (e.g. `JWT`, `at+jwt`) | Optional, ignored by JWS itself |
| `cty` | **Content Type** of the payload | Used for nested JWTs (cty=`JWT`) |
| `crit` | **Critical** extensions — array of header param names that MUST be understood | MUST be in the Protected Header. Standard params MUST NOT be listed |

`alg` is the only **mandatory** parameter. Everything else is contextual.

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "2026-04-rotation"
}
```

## Producing a JWS (§5.1)

1. Build the payload bytes.
2. Compute `BASE64URL(payload)`.
3. Build the Protected Header JSON; compute `BASE64URL(UTF8(header))`.
4. Compute the **Signing Input** — `ASCII(b64u_header || '.' || b64u_payload)`.
5. Sign that input with the algorithm in `alg` and the appropriate key.
6. Compute `BASE64URL(signature)`.
7. Concatenate with dots → Compact form (or assemble a JSON object — see [[JWT JO 03 - JWS Serializations]]).

## Validating a JWS (§5.2)

1. Split / parse the serialization.
2. Base64url-decode the protected header; assert it's valid UTF-8 JSON.
3. Build the JOSE Header (Compact: protected only; JSON: union of protected + unprotected, names must be unique).
4. Verify the implementation **understands every param in `crit`** — fail otherwise.
5. Base64url-decode the payload and signature.
6. Recompute the Signing Input from the **received** encoded header + payload bytes (do not re-canonicalize JSON).
7. Verify the signature with `alg` + the resolved key.

> "Even if a JWS can be successfully validated, unless the algorithm(s) used in the JWS are acceptable to the application, it SHOULD consider the JWS to be invalid." — RFC 7515 §5.2

This sentence is the entire rationale for **algorithm allow-listing** ([[JWT SE 02 - Algorithm Confusion]]).

## Protected vs Unprotected headers

- **Protected Header** — covered by the signature. Compact form supports only this.
- **Unprotected Header** — JSON serialization only; **not signed**. Use only for hints (e.g. `kid`) where forgery is acceptable.

⚠️ **Footguns:**
- **Verify with `alg` from a trusted policy, not from the token.** Otherwise an attacker swaps `RS256 → HS256` and signs with your public key as the HMAC secret. See [[JWT SE 02 - Algorithm Confusion]].
- **`crit` parameters that you don't recognize → reject the JWS.** Don't ignore.
- **Re-encoding the JSON header before verifying breaks the signature.** Always feed the verifier the **received** base64url bytes.
- **`jwk`/`jku`/`x5u`/`kid` in the header are attacker input** — never resolve them without a trust anchor. See [[JWT SE 03 - Key Management]].
- **`x5c` uses standard base64**, not base64url. (Yes, really.)
- **HMAC keys (`HS*`)** must be strong, random, and at least the hash length — see [[JWT AL 01 - HMAC]].

## Algorithm-family pointers

| Family | Notes | Note |
|---|---|---|
| `HS256/384/512` | Symmetric HMAC-SHA-2 | [[JWT AL 01 - HMAC]] |
| `RS256/384/512` | RSASSA-PKCS1-v1_5 | [[JWT AL 02 - RSA Signatures]] |
| `PS256/384/512` | RSASSA-PSS (preferred over RS*) | [[JWT AL 02 - RSA Signatures]] |
| `ES256/384/512` | ECDSA on P-256 / P-384 / P-521 | [[JWT AL 03 - ECDSA]] |
| `EdDSA` | Ed25519 / Ed448 (RFC 8037) | [[JWT AL 04 - EdDSA]] |
| `none` | **Unsecured** — must be explicitly allowed | [[JWT AL 05 - none Algorithm]] |

💡 **Takeaway:** A JWS is *(header, payload, sig over base64url-encoded header.payload)*. The `alg` declares **what was used**; your verifier decides **what is acceptable** — and that decision is where almost every JWS bug lives.
