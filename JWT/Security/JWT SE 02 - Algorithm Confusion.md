# SE 02 — Algorithm Confusion

🔑 **One-line: never let the token's `alg` header decide how you verify it — pin the algorithm on the verifier side or the attacker forges tokens at will.**

## Threat

Two classic, library-level attacks share the same root cause: the verifier reads `alg` from the (unauthenticated) header, then uses whatever the attacker put there.

**Attack 1 — `alg=none` bypass.** RFC 8725 §2.1 calls this out directly: an attacker sets `alg` to `"none"`, drops the signature, and some libraries treat the token as valid because the spec lists `none` as a registered algorithm. See [[JWT AL 05 - none Algorithm]].

**Attack 2 — RS256 → HS256 key confusion.** The server expects RS256 (asymmetric, see [[JWT AL 02 - RSA Signatures]]) and passes the RSA *public* key to the verify function. The attacker forges a token using HS256 and HMACs it with the public key as the secret. From the Auth0 writeup:

> "If a server is expecting a token signed with RSA, but actually receives a token signed with HMAC, it will think the public key is actually an HMAC secret key."

The attacker code is one line:

```python
forged = jwt.encode(payload, public_key_pem, algorithm="HS256")
```

The over-the-wire request looks completely innocuous:

```http
GET /api/admin HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.<hmac-of-public-key>
```

Or even more brazenly with `alg=none`:

```http
GET /api/admin HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiJ9.
```

The Auth0 disclosure summarised the root cause:

> "Attackers control the choice of algorithm" because the verification function reads `alg` from the untrusted header before validation completes.

## Mitigation

RFC 8725 §3.1 is unambiguous:

> "Libraries MUST enable the caller to specify a supported set of algorithms and MUST NOT use any other algorithms when performing cryptographic operations."

And §3.2 on `none`:

> The `none` algorithm "is acceptable only when the JWT is cryptographically protected by other means... Libraries should not generate or consume `none` JWTs without explicit caller request."

In practice: pin the algorithm explicitly, and bind each key to one algorithm via `kid`.

```python
import jwt  # see [[JWT PR 01 - PyJWT]]

claims = jwt.decode(
    token,
    key=public_key,
    algorithms=["RS256"],  # whitelist — never read alg from the header
    audience="https://api.example.com",
    issuer="https://auth.example.com",
)
```

```javascript
// see [[JWT PR 03 - jsonwebtoken Node]]
const claims = jwt.verify(token, publicKey, {
  algorithms: ["RS256"],   // mandatory; never default
  audience: "https://api.example.com",
  issuer:   "https://auth.example.com",
});
```

Defence-in-depth: each key in your [[JWT JO 05 - JWK]] set carries an `alg` field, and the verifier should refuse a `kid` whose stored algorithm doesn't match the algorithm it was about to use.

⚠️ Passing `algorithms=None` (PyJWT) or omitting `algorithms` (jsonwebtoken pre-9.0) is the bug. Older `jsonwebtoken` defaulted to verifying with whatever `alg` the token claimed — that is exactly the vulnerability.

⚠️ A "fix" that only blocks `alg=none` does **not** prevent the RS256 → HS256 attack. You must whitelist the *expected* algorithm, not blacklist the bad ones.

⚠️ If you operate both HMAC and RSA keys, never let the same key bytes serve both. Public keys are public — an attacker who reads them off your `/.well-known/jwks.json` ([[JWT SE 03 - Key Management]]) gets an HMAC secret for free if your verifier accepts HS256.

The deeper structural mitigation (Auth0's recommendation, echoed in RFC 8725 §3.1):

- Bind `kid` → algorithm at registration time. The verifier looks up the key by `kid`, reads the *server-side* algorithm, and uses that.
- Refuse to verify if the token's `alg` doesn't match the key's `alg`.
- Maintain separate keys for HMAC and RSA — never reuse public-key bytes as a symmetric secret.

Cross-references:

- The header field that carries `alg`: [[JWT JO 02 - JWS]].
- Algorithm definitions: [[JWT JO 06 - JWA]], [[JWT AL 01 - HMAC]], [[JWT AL 02 - RSA Signatures]], [[JWT AL 05 - none Algorithm]].
- The broader BCP: [[JWT SE 01 - RFC 8725 BCP]].
- Why "verified" ≠ "validated": [[JWT FN 05 - Validating vs Verifying]].
- Token shape: [[JWT FN 01 - What Is a JWT]].

💡 **Takeaway:** the `alg` header is attacker-controlled — never trust it; pin algorithms server-side and bind each key to exactly one algorithm.
