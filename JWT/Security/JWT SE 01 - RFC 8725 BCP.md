# SE 01 — RFC 8725 BCP

🔑 **One-line: RFC 8725 is the IETF "JSON Web Token Best Current Practices" — the canonical checklist of JWT threats and the rules every implementer must follow.**

## Threat

JWTs are deceptively simple. The signed/encrypted token format hides a forest of footguns: algorithm selection is in the (untrusted) header, claims are application-specific, and libraries historically defaulted to permissive behaviour. RFC 8725 collects the documented attacks and turns them into MUSTs.

The headline threats it catalogues:

- **2.1 Weak Signatures / Insufficient Validation** — `alg=none` bypass and RS256 → HS256 key confusion. See [[JWT SE 02 - Algorithm Confusion]] and [[JWT AL 05 - none Algorithm]].
- **2.2 Weak Symmetric Keys** — human-memorable passwords used as HMAC secrets. Vulnerable to offline brute-force once a token leaks.
- **2.3 Incorrect Composition of Encryption and Signature** — JWEs that decrypt successfully without validating the inner JWS signature.
- **2.4 Plaintext Leakage via Ciphertext Length** — compression before encryption leaks information about the plaintext.
- **2.5 Insecure Elliptic Curve Encryption** — failing to validate ECDH-ES public key inputs lets an attacker recover the private key.
- **2.6 Multiplicity of JSON Encodings** — ambiguous UTF-16/UTF-32 encodings used to bypass validation checks.
- **2.7 Substitution Attacks** — a token issued for recipient A is replayed at recipient B.
- **2.8 Cross-JWT Confusion** — a token issued for one purpose is accepted by a verifier expecting a different purpose. See [[JWT SE 06 - Cross-JWT Confusion]].
- **2.9 Indirect Attacks on the Server** — `kid`, `jku`, `x5u` used as injection / SSRF vectors. See [[JWT SE 03 - Key Management]].

```http
GET /api/admin HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiJ9.
```

That trailing dot is the empty signature on an `alg=none` token — RFC 8725 §2.1 exists because libraries used to accept it.

## Mitigation

The Best Practices in §3 are non-negotiable. Treat them as a checklist for any JWT-touching code review.

> §3.1: "Libraries MUST enable the caller to specify a supported set of algorithms and MUST NOT use any other algorithms when performing cryptographic operations."

> §3.2: The `none` algorithm "is acceptable only when the JWT is cryptographically protected by other means."

> §3.5: "Human-memorizable passwords MUST NOT be directly used as the key to a keyed-MAC algorithm such as HS256."

> §3.9 (Validate Audience): "if the audience value is not present or not associated with the recipient, it MUST reject the JWT."

> §3.11 (Use Explicit Typing): use `typ` to disambiguate JWT kinds — see [[JWT SE 06 - Cross-JWT Confusion]].

> §3.12 (Mutually Exclusive Validation Rules): validation rules for different JWT kinds "MUST be mutually exclusive."

The minimum viable verifier — every parameter is mandatory:

```python
import jwt  # PyJWT — see [[JWT PR 01 - PyJWT]]

claims = jwt.decode(
    token,
    key=public_key,
    algorithms=["RS256"],          # §3.1 — pin the algorithm
    audience="https://api.example.com",  # §3.9 — required when multi-recipient
    issuer="https://auth.example.com",   # §3.8 — validate issuer
    options={
        "require": ["exp", "iat", "iss", "aud"],
        "verify_signature": True,
        "verify_exp": True,
        "verify_iat": True,
    },
)
```

The Node equivalent lives in [[JWT PR 03 - jsonwebtoken Node]]; the underlying claims and signing primitives are in [[JWT FN 03 - Registered Claims]], [[JWT JO 02 - JWS]], and [[JWT JO 06 - JWA]].

⚠️ The biggest footgun in §3.1: **never** let the verifier read `alg` from the token to choose the verification algorithm. Whitelist on the verifier side, key off the `kid`, and reject anything that doesn't match. See [[JWT FN 05 - Validating vs Verifying]].

⚠️ §2.9 / §3.10: do not feed `kid`, `jku`, or `x5u` directly into a database query, filesystem path, or URL fetch. They are attacker-controlled. Treat them as untrusted strings; for `jku` / `x5u`, match against an allow-list and never send cookies on the fetch. Detailed in [[JWT SE 03 - Key Management]].

⚠️ §2.7 substitution attacks bite even when every signature is valid. The fix is `aud` and explicit `typ` — see [[JWT SE 05 - Validation Pitfalls]].

The matching algorithm and key primitives:

- HMAC: [[JWT AL 01 - HMAC]] — symmetric, requires high-entropy shared secret.
- RSA: [[JWT AL 02 - RSA Signatures]] — asymmetric, public key is widely shared (which is exactly why §3.1 matters).
- The wire format that carries `alg` and `kid`: [[JWT JO 02 - JWS]].
- The key distribution format: [[JWT JO 05 - JWK]].
- What the token actually is: [[JWT FN 01 - What Is a JWT]].

The two main JWT use cases that depend on every BCP rule below:

- [[JWT UC 02 - OAuth Access Tokens]] — short-lived, audience-scoped, `aud` validation is load-bearing.
- [[JWT UC 03 - OIDC ID Tokens]] — `iss`, `aud`, `nonce` validation is mandated by OIDC, on top of RFC 8725.

Companion notes that drill into specific BCP sections:

- §2.1, §3.1, §3.2 → [[JWT SE 02 - Algorithm Confusion]]
- §3.10, RFC 7515 §4.1.2/4.1.5 → [[JWT SE 03 - Key Management]]
- §2 (expiration / replay) → [[JWT SE 04 - Expiration and Revocation]]
- §3.8, §3.9, §3.10, §3.11 → [[JWT SE 05 - Validation Pitfalls]]
- §2.7, §2.8, §3.11, §3.12 → [[JWT SE 06 - Cross-JWT Confusion]]

💡 **Takeaway:** RFC 8725 is short, prescriptive, and free — read it once, then audit every JWT verifier you own against §3.1 through §3.12.
