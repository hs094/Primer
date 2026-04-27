# SE 05 — Validation Pitfalls

🔑 **One-line: signature verification is necessary but not sufficient — you must also validate `iss`, `aud`, `typ`, and the trust path of every claim you act on.**

## Threat

A valid signature only proves "someone with the signing key produced this token." It does **not** prove the token was issued for *you*, for *this purpose*, or that the claims inside are trustworthy. RFC 8725 §3.8–§3.11 enumerate the four validation steps libraries don't do for you.

**Pitfall 1 — Skipping issuer validation (§3.8).** A signature from any issuer in your JWKS-trust-store passes verification. If you trust both `auth.example.com` and `partner.example.com`, a partner-issued token impersonates an `example.com` user unless you check `iss`.

> §3.8: applications must validate that "the issuer is a trusted one" and that the cryptographic keys "actually belong to the issuer."

**Pitfall 2 — Skipping audience validation (§3.9).** RFC 7519 §4.1.3 is the original requirement:

> "Each principal intended to process the JWT MUST identify itself with a value in the audience claim. If the principal processing the claim does not identify itself with a value in the 'aud' claim when this claim is present, then the JWT MUST be rejected."

RFC 8725 §3.9 hardens it:

> "if the audience value is not present or not associated with the recipient, it MUST reject the JWT."

The substitution attack (RFC 8725 §2.7): the same SSO issuer mints tokens for `billing-api.example.com` and `admin-api.example.com`. A user with billing access grabs their token from devtools and replays it at the admin API. Same issuer, same signature, same `sub` — different `aud`. Without audience checking, the admin API accepts it.

```http
GET /admin/users HTTP/1.1
Host: admin-api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyXzQyIiwiYXVkIjoiaHR0cHM6Ly9iaWxsaW5nLWFwaS5leGFtcGxlLmNvbSJ9.sig
```

Decoded `aud`: `https://billing-api.example.com`. Same signature; wrong recipient. Reject.

**Pitfall 3 — Implicit typing (§3.11).** Without `typ`, a refresh token, an ID token, and a custom application JWT all look identical to a verifier. See [[JWT SE 06 - Cross-JWT Confusion]].

> §3.11: "JWT types can be specified via the `typ` header parameter to prevent confusion between different JWT kinds. For new JWT applications, the use of explicit typing is RECOMMENDED."

**Pitfall 4 — Trusting received claims (§3.10).** Claims like `email`, `roles`, `is_admin` in the JWT are only as trustworthy as the issuer's claim source. If the issuer doesn't verify the email, neither can you.

> §3.10 (Do Not Trust Received Claims): validate `kid` to prevent SQL/LDAP injection; protect against SSRF when following `jku` / `x5u`; "matching the URL to a whitelist of allowed locations and ensuring no cookies are sent."

## Mitigation

Validate all four explicitly, on every request. Library defaults are not enough.

```python
import jwt  # [[JWT PR 01 - PyJWT]]

claims = jwt.decode(
    token,
    key=public_key,
    algorithms=["RS256"],                      # §3.1 — see [[JWT SE 02 - Algorithm Confusion]]
    audience="https://api.example.com",        # §3.9 — required
    issuer="https://auth.example.com",         # §3.8 — required
    options={
        "require": ["exp", "iat", "iss", "aud", "typ"],
        "verify_aud": True,
        "verify_iss": True,
    },
)

# §3.11 — explicit typing
header = jwt.get_unverified_header(token)
if header.get("typ") != "at+jwt":              # access token type — RFC 9068
    raise AuthError("wrong JWT type")
```

```javascript
// [[JWT PR 03 - jsonwebtoken Node]]
const claims = jwt.verify(token, publicKey, {
  algorithms: ["RS256"],
  audience: "https://api.example.com",
  issuer:   "https://auth.example.com",
});
const { typ } = jwt.decode(token, { complete: true }).header;
if (typ !== "at+jwt") throw new AuthError("wrong JWT type");
```

The mental model: signature verification proves *integrity*; the validation steps above prove *applicability*. RFC 8725 §3.8–§3.11 is the spec for "applicability."

For multi-tenant / multi-audience APIs, validate `aud` against an allow-list — `aud` may be a string or array (RFC 7519 §4.1.3):

```python
ALLOWED_AUDIENCES = {"https://api.example.com", "https://api.example.com/v2"}

aud_values = {claims["aud"]} if isinstance(claims["aud"], str) else set(claims["aud"])
if not aud_values & ALLOWED_AUDIENCES:
    raise AuthError("audience mismatch")
```

For claim trust (§3.10), the rule is:

- **Identity claims** (`sub`, `email`, `email_verified`) — trust only if the issuer verifies them. OIDC ([[JWT UC 03 - OIDC ID Tokens]]) requires `email_verified` as a separate boolean for exactly this reason.
- **Authorisation claims** (`scope`, `roles`) — trust the issuer's authoritative source. If you re-derive permissions, treat the JWT claim as a hint, not a source of truth.
- **Lookup keys** (`kid`, `sub`) — sanitise before using in queries. Parameterised SQL / LDAP escape, no string concatenation.

⚠️ Most JWT libraries do **not** validate `iss` or `aud` unless you pass them as parameters. PyJWT requires `audience=` and `issuer=`; jsonwebtoken requires `audience` and `issuer` in the options. Omitting them silently disables the check. See [[JWT FN 05 - Validating vs Verifying]].

⚠️ Don't validate `aud` with `aud == EXPECTED` when the spec allows `aud` to be an array. Use set membership.

⚠️ RFC 8725 §3.11 on nested JWTs: the `typ` "MUST be present in the inner JWT" of a nested JWT. Easy to forget when wrapping a JWS in a JWE.

⚠️ Treat any string from the JWT that you'll use as a database key, URL, file path, or shell argument as **untrusted**. Signature validity says nothing about claim *content*. The issuer is trusted to mint tokens, not to sanitise your inputs.

Cross-references:

- BCP overview: [[JWT SE 01 - RFC 8725 BCP]].
- Cross-JWT specifics: [[JWT SE 06 - Cross-JWT Confusion]].
- Time-claim validation: [[JWT SE 04 - Expiration and Revocation]].
- The claims themselves: [[JWT FN 03 - Registered Claims]].
- Why "verified" ≠ "validated": [[JWT FN 05 - Validating vs Verifying]].
- Token format: [[JWT FN 01 - What Is a JWT]], [[JWT JO 02 - JWS]].
- Use cases that require these checks: [[JWT UC 02 - OAuth Access Tokens]], [[JWT UC 03 - OIDC ID Tokens]].

💡 **Takeaway:** verify the signature, then validate `iss`, `aud`, and `typ` — and treat every claim's *content* as untrusted input.
