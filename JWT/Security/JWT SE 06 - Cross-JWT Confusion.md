# SE 06 — Cross-JWT Confusion

🔑 **One-line: a JWT minted for purpose A must be unusable as a JWT for purpose B — distinguish them with `typ`, `aud`, distinct keys, or all three.**

## Threat

When the same issuer (or the same key) signs JWTs for multiple purposes — access tokens, ID tokens, password-reset tokens, webhook signatures — verifiers downstream can mistake one for another. RFC 8725 calls these out as separate but related problems:

> §2.7 (Substitution Attacks): legitimate tokens "intended for one recipient may be misused at another recipient."

> §2.8 (Cross-JWT Confusion): JWTs issued for one protocol risk "being subverted and used for another, particularly as JWT deployment proliferates across diverse applications."

The combinatorics are nasty: with N JWT types signed by the same key, every verifier accidentally accepts every other type unless validation rules differ.

Concrete attacks:

- **OAuth access token replayed as OIDC ID token** ([[JWT UC 02 - OAuth Access Tokens]] vs [[JWT UC 03 - OIDC ID Tokens]]). Same `iss`, same signature, different intent. The relying party trusts `email` from the ID token; the access token also has an `email` claim; without `typ` or `aud` checks, the access token logs the user in.
- **Password-reset token replayed as session token.** Issuer mints a 10-minute reset JWT with `{ "sub": "user_42", "purpose": "reset" }`. A verifier looking for `sub` accepts it as a login.
- **Webhook signature replayed as auth token.** Same key signs outgoing webhook payloads and inbound API tokens. An attacker captures a webhook delivery, extracts the JWS, and presents it as an API credential.

```http
POST /oidc/userinfo HTTP/1.1
Host: rp.example.com
Content-Type: application/json

{ "id_token": "eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyXzQyIiwiYXVkIjoid2ViLWFwcCIsImVtYWlsIjoidmljdGltQGV4YW1wbGUuY29tIn0.sig" }
```

That payload is actually an *access token* — not an ID token — but the relying party didn't check `typ`. The signature is valid, `iss` matches, `sub` is present, and the verifier signs the user in as `victim@example.com`.

## Mitigation

RFC 8725 §3.11 (Use Explicit Typing):

> "JWT types can be specified via the `typ` header parameter to prevent confusion between different JWT kinds... The `application/` prefix should be omitted; for example, Security Event Tokens use `secevent+jwt`. For nested JWTs, explicit typing MUST be present in the inner JWT."

RFC 8725 §3.12 (Use Mutually Exclusive Validation Rules) — the structural rule:

> Applications issuing multiple JWT types must employ validation rules that are "mutually exclusive, rejecting JWTs of the wrong kind. For new JWT applications, the use of explicit typing is RECOMMENDED."

The strategies §3.12 lists, ranked by strength:

1. **Different signing keys per JWT type.** Strongest. A misused token fails signature verification.
2. **Distinct `typ` header values.** The standardised approach. RFC 9068 defines `at+jwt` for OAuth access tokens; OIDC ID tokens are JWT but conventionally lack a `typ` (which itself is now considered a wart).
3. **Distinct `aud` values per recipient.** Required by RFC 8725 §3.9 anyway.
4. **Distinct issuers (`iss`).** Helpful when the trust model already separates them.
5. **Different required claim sets.** Weakest — easy to bypass by adding extra claims.

Pick at least two and enforce all of them.

```python
# Issuer side — set typ explicitly
header = {"alg": "RS256", "typ": "at+jwt", "kid": current_kid}
payload = {
    "iss": "https://auth.example.com",
    "sub": user.id,
    "aud": "https://api.example.com",
    "exp": now + 600,
    "iat": now,
    "jti": uuid4().hex,
}
access_token = jwt.encode(payload, signing_key, algorithm="RS256", headers=header)
```

```python
# Verifier side — pin typ and aud
unverified = jwt.get_unverified_header(token)
if unverified.get("typ") != "at+jwt":
    raise AuthError("not an access token")

claims = jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],
    audience="https://api.example.com",   # §3.9 — recipient identity
    issuer="https://auth.example.com",    # §3.8 — trusted issuer
    options={"require": ["exp", "iat", "iss", "aud", "jti"]},
)
```

For systems where you control both ends, the strongest move is **separate signing keys per JWT type**:

- `kid: "access-2024"` — used only to sign access tokens.
- `kid: "reset-2024"` — used only to sign password-reset tokens.
- `kid: "webhook-2024"` — used only to sign outbound webhooks.

The verifier picks the key by purpose, not by what the token claims:

```python
# Wrong: verifier uses whichever key the token's kid points at
key = jwks.get_signing_key(header["kid"])

# Right: verifier knows which purpose it's verifying for
key = jwks.get_signing_key_for_purpose("access-token")
```

A misused token now fails signature verification immediately. RFC 8725 §3.12 lists this as the strongest option.

⚠️ §3.11 says explicit `typ` is RECOMMENDED for new applications — but historically, OIDC ID tokens omitted `typ`, so plain `typ: "JWT"` (or no `typ`) does **not** distinguish ID tokens from access tokens. Use `at+jwt` for OAuth access tokens (RFC 9068) and rely on `aud` to distinguish ID tokens.

⚠️ Re-using one key across JWT purposes saves a JWKS entry and costs you a §3.12 violation. The "savings" disappears the first time someone files a CVE.

⚠️ Nested JWTs (a JWS inside a JWE) need `typ` on the **inner** JWT — RFC 8725 §3.11 makes this explicit. Easy to miss because the outer envelope is already typed.

⚠️ "We use `aud` to differentiate" is fine for substitution attacks (§2.7) but does **not** protect against an attacker who legitimately holds tokens for both audiences. Combine `aud` with `typ` or distinct keys.

Cross-references:

- BCP overview: [[JWT SE 01 - RFC 8725 BCP]].
- The validation steps that detect cross-JWT confusion: [[JWT SE 05 - Validation Pitfalls]].
- Algorithm-level cousin attack: [[JWT SE 02 - Algorithm Confusion]].
- Key/JWKS layout for per-purpose keys: [[JWT SE 03 - Key Management]], [[JWT JO 05 - JWK]].
- Token wire format and headers: [[JWT JO 02 - JWS]], [[JWT FN 01 - What Is a JWT]].
- Algorithm registry: [[JWT JO 06 - JWA]].
- Concrete examples of distinct JWT kinds: [[JWT UC 02 - OAuth Access Tokens]], [[JWT UC 03 - OIDC ID Tokens]].
- Library-level enforcement: [[JWT PR 01 - PyJWT]], [[JWT PR 03 - jsonwebtoken Node]].

💡 **Takeaway:** never let one JWT pass for another — combine explicit `typ`, scoped `aud`, and (best) per-purpose signing keys.
