# FN 01 — What Is a JWT

🔑 **A JWT is a compact, URL-safe JSON object whose claims are signed (or encrypted) so a verifier can trust them without calling back to the issuer.**

## Definition (RFC 7519 §1)

> JSON Web Token (JWT) is a compact, URL-safe means of representing claims to be transferred between two parties. The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure.

Three things in that sentence matter:

- **Claims** — name/value statements about a subject.
- **JSON object** — the payload, full stop. Not XML, not a binary blob.
- **JWS or JWE** — the wrapper that makes claims tamper-evident (signed) or confidential (encrypted). See [[JWT JO 02 - JWS]] and [[JWT JO 04 - JWE]].

## Shape

```text
xxxxx.yyyyy.zzzzz
header . payload . signature
```

Each segment is base64url-encoded. Periods are literal. See [[JWT FN 02 - Anatomy and Encoding]].

## Why JWTs Exist

From jwt.io's introduction, the design goals are:

- **Self-contained.** Claims travel inside the token; verifiers don't need a session lookup.
- **Compact.** Smaller than SAML, easy to put in a URL, header, or POST body.
- **Standardized signing.** HMAC, RSA, or ECDSA via [[JWT JO 06 - JWA]].

## Authorization Flow (jwt.io)

1. Client logs in, gets a JWT from the auth server.
2. Client sends it on every request:
   ```text
   Authorization: Bearer <token>
   ```
3. Server verifies the signature and validates claims.
4. If both pass, the request is authorized — no DB hit required.

This is the canonical [[JWT UC 01 - API Authentication]] pattern, and the substrate for [[JWT UC 02 - OAuth Access Tokens]] and [[JWT UC 03 - OIDC ID Tokens]].

## Two Common Uses

- **Authorization** — the bearer-token pattern above. Most common.
- **Information exchange** — pass a signed payload between services so the receiver can trust origin and integrity.

## What a JWT Is NOT

- Not a session — it's a credential the server stamps once and trusts on sight.
- Not encrypted by default — a signed JWT (JWS) is *readable by anyone*. Only [[JWT JO 04 - JWE]] hides contents.
- Not revocable on its own — once issued, a signed JWT is valid until `exp`. See [[JWT SE 04 - Expiration and Revocation]].

⚠️ **A signed JWT is not confidential.** Anyone with the token can base64-decode the payload and read every claim. Don't put passwords, PII, or secrets in there.

⚠️ **"Stateless" cuts both ways.** No DB lookup means no easy logout. Plan revocation up front, not after launch.

## Minimal Example (RFC 7519 §3.1)

```json
{ "typ": "JWT", "alg": "HS256" }
```

```json
{
  "iss": "joe",
  "exp": 1300819380,
  "http://example.com/is_root": true
}
```

Resulting compact serialization:

```text
eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ.dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Paste it into [[JWT PR 04 - jwt.io Debugger]] to see each part decoded.

## Where It Sits in the Stack

- **Format** — [[JWT FN 02 - Anatomy and Encoding]]
- **Claims taxonomy** — [[JWT FN 03 - Registered Claims]], [[JWT FN 04 - Public and Private Claims]]
- **Trust model** — [[JWT FN 05 - Validating vs Verifying]]
- **Container family** — [[JWT JO 01 - JOSE Overview]]

💡 **Takeaway:** JWT = signed JSON claims in a `header.payload.signature` envelope. Self-contained credentials, not sessions, not secrets.
