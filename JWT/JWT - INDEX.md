# JWT Knowledge Pack

Refresher vault mirroring the Auth0 JWT Handbook structure, sourced from the IETF RFCs (7515/7516/7517/7518/7519/8037/8725, 6749/6750/9068), [jwt.io](https://jwt.io/introduction), and library docs (PyJWT, joserfc, node-jsonwebtoken). Code-first, terse — for re-reading, not first-time learning.

🔑 **Why JWT:** a compact, URL-safe, self-contained token for transmitting signed (and optionally encrypted) JSON claims between parties. RFC 7519 defines the claim set; the JOSE family (JWS/JWE/JWK/JWA) defines the cryptographic envelope. Use it for stateless API auth, OAuth access tokens, OIDC ID tokens — and validate every claim every time.

## Foundations

| # | Note | What it covers |
|---|---|---|
| 01 | [[JWT FN 01 - What Is a JWT]] | RFC 7519 definition, shape, when to use |
| 02 | [[JWT FN 02 - Anatomy and Encoding]] | header.payload.signature, base64url, RFC §3.1 worked example |
| 03 | [[JWT FN 03 - Registered Claims]] | `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti` |
| 04 | [[JWT FN 04 - Public and Private Claims]] | IANA-registered vs collision-resistant URI vs private claims |
| 05 | [[JWT FN 05 - Validating vs Verifying]] | RFC §7.2 — verify signature, then validate claims |

## JOSE

| # | Note | What it covers |
|---|---|---|
| 01 | [[JWT JO 01 - JOSE Overview]] | The four-spec family (JWS/JWE/JWK/JWA) and how JWT sits in it |
| 02 | [[JWT JO 02 - JWS]] | RFC 7515 — signing/MAC, all `alg`/`kid`/`jku`/`jwk`/`x5*` header parameters |
| 03 | [[JWT JO 03 - JWS Serializations]] | Compact vs JSON General vs JSON Flattened |
| 04 | [[JWT JO 04 - JWE]] | RFC 7516 — five-part compact, key wrap vs direct, ciphertext+tag |
| 05 | [[JWT JO 05 - JWK]] | RFC 7517 — `kty`, JWK Set, `kid`/`use`/`key_ops`/`alg` |
| 06 | [[JWT JO 06 - JWA]] | RFC 7518 — algorithm registry: `HS*`, `RS*`, `ES*`, `PS*`, `A*KW`, `A*GCM`, `RSA-OAEP`, `ECDH-ES` |

## Algorithms

| # | Note | What it covers |
|---|---|---|
| 01 | [[JWT AL 01 - HMAC]] | `HS256`/`HS384`/`HS512` — symmetric, shared secret |
| 02 | [[JWT AL 02 - RSA Signatures]] | `RS*` (PKCS#1 v1.5) and `PS*` (RSASSA-PSS) |
| 03 | [[JWT AL 03 - ECDSA]] | `ES256` (P-256), `ES384`, `ES512` (P-521) |
| 04 | [[JWT AL 04 - EdDSA]] | RFC 8037 — Ed25519, Ed448 |
| 05 | [[JWT AL 05 - none Algorithm]] | Unsecured JWT, why it's banned in practice |

## Security

| # | Note | What it covers |
|---|---|---|
| 01 | [[JWT SE 01 - RFC 8725 BCP]] | Threat catalog and 12 best practices overview |
| 02 | [[JWT SE 02 - Algorithm Confusion]] | `alg=none` bypass, `RS256`→`HS256` key confusion |
| 03 | [[JWT SE 03 - Key Management]] | `kid`/`jku`/`x5u` resolution, JWKS, SSRF, rotation |
| 04 | [[JWT SE 04 - Expiration and Revocation]] | Short TTLs, `jti` blocklists, refresh tokens |
| 05 | [[JWT SE 05 - Validation Pitfalls]] | Issuer/audience/typ checks, claim trust |
| 06 | [[JWT SE 06 - Cross-JWT Confusion]] | Substitution attacks, mutually exclusive validation rules |

## Practice

| # | Note | What it covers |
|---|---|---|
| 01 | [[JWT PR 01 - PyJWT]] | `jwt.encode`/`jwt.decode`, `PyJWKClient`, exceptions |
| 02 | [[JWT PR 02 - joserfc Python]] | Modern Python JOSE library — JWS/JWE/JWK/JWT |
| 03 | [[JWT PR 03 - jsonwebtoken Node]] | `sign`/`verify`/`decode`, options, async patterns |
| 04 | [[JWT PR 04 - jwt.io Debugger]] | The web debugger, library catalog, browser extension |

## Use Cases

| # | Note | What it covers |
|---|---|---|
| 01 | [[JWT UC 01 - API Authentication]] | RFC 6750 Bearer tokens, stateless API auth |
| 02 | [[JWT UC 02 - OAuth Access Tokens]] | RFC 9068 JWT Profile — `typ: at+jwt`, mandatory claims |
| 03 | [[JWT UC 03 - OIDC ID Tokens]] | OpenID Connect Core §2/§3.1.3.7 — claims, validation steps |
| 04 | [[JWT UC 04 - Storage and Transport]] | Cookie vs Authorization vs localStorage; SameSite, HttpOnly |

💡 **Takeaway:** start at [[JWT FN 01 - What Is a JWT]] / [[JWT FN 03 - Registered Claims]] for the basics, [[JWT JO 02 - JWS]] / [[JWT JO 06 - JWA]] for the wire format, [[JWT AL 02 - RSA Signatures]] / [[JWT AL 03 - ECDSA]] for the math you'll actually pick, [[JWT SE 01 - RFC 8725 BCP]] before shipping anything, and [[JWT PR 01 - PyJWT]] when writing Python.
