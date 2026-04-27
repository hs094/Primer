# SE 03 — Key Management

🔑 **One-line: `kid`, `jku`, and `x5u` are attacker-controlled hints — never feed them straight into queries, file paths, or HTTP fetches.**

## Threat

JWS headers carry several "where is the key?" hints. RFC 7515 §4.1.4 says it plainly for `kid`:

> "The 'kid' (key ID) Header Parameter is a hint indicating which key was used to secure the JWS."

The dangerous ones are the URL-bearing siblings in §4.1.2 (`jku`) and §4.1.5 (`x5u`):

> §4.1.2: "The 'jku' (JWK Set URL) Header Parameter is a URI that refers to a resource for a set of JSON-encoded public keys, one of which corresponds to the key used to digitally sign the JWS."

> §4.1.5: "The 'x5u' (X.509 URL) Header Parameter is a URI that refers to a resource for the X.509 public key certificate or certificate chain corresponding to the key used to digitally sign the JWS."

All of these live in the JOSE header ([[JWT JO 02 - JWS]]) and are **fully controlled by whoever crafts the token**. RFC 8725 §2.9 and §3.10 spell out the consequences:

- **Injection via `kid`** — used as a lookup key into SQL, LDAP, or the filesystem. `kid = "../../../dev/null"` makes the verifier read empty bytes; a SQL-injecting `kid` returns an attacker-chosen key.
- **SSRF via `jku` / `x5u`** — the verifier dutifully fetches the URL the attacker supplied, hitting internal-only services or attacker-hosted JWK sets.

```http
POST /api/auth/verify HTTP/1.1
Host: api.example.com
Content-Type: application/jwt

eyJhbGciOiJSUzI1NiIsImtpZCI6Ii4uLy4uLy4uL2Rldi9udWxsIiwianl1IjoiaHR0cDovLzE2OS4yNTQuMTY5LjI1NC9sYXRlc3QvbWV0YS1kYXRhL2lhbS9zZWN1cml0eS1jcmVkZW50aWFscy8ifQ...
```

That decoded header:

```json
{
  "alg": "RS256",
  "kid": "../../../dev/null",
  "jku": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
}
```

A naive verifier that does `requests.get(header["jku"])` just exfiltrated EC2 IAM credentials. A naive verifier that does `open(f"keys/{header['kid']}.pem")` just read `/dev/null` and treated empty bytes as a key. RFC 8725 §3.10 categorises both as "Indirect Attacks on the Server."

There is also the trust-store attack: an attacker who can write to your JWK Set endpoint, or MITM its delivery, replaces the issuer's public key with theirs and forges every subsequent token. RFC 7515 §4.1.2 and RFC 7517 both require TLS for this reason:

> "The protocol used to acquire the resource MUST provide integrity protection; an HTTP GET request to retrieve the JWK Set MUST use Transport Layer Security (TLS)."

## Mitigation

Treat header URLs and identifiers as **untrusted input**. The verifier — not the token — decides where keys come from.

```python
# Pin the JWKS endpoint server-side. Never use header["jku"].
JWKS_URL = "https://auth.example.com/.well-known/jwks.json"

jwks_client = jwt.PyJWKClient(JWKS_URL)  # cached, TLS-validated

unverified_header = jwt.get_unverified_header(token)
kid = unverified_header.get("kid")

# kid is a hint — validate it against the JWKS we control.
signing_key = jwks_client.get_signing_key(kid)  # raises if unknown

claims = jwt.decode(
    token,
    signing_key.key,
    algorithms=["RS256"],
    audience="https://api.example.com",
    issuer="https://auth.example.com",
)
```

The structural rules from RFC 7517 ([[JWT JO 05 - JWK]]) and RFC 8725 §3.10:

- **Pin the JWKS URL server-side.** It is configuration, not an input. A `BaseSettings` field with the issuer's well-known endpoint, full stop.
- **Cache the JWKS** with a sensible TTL (5–60 minutes is typical) and refresh on `kid` miss.
- **Always use TLS** for JWKS retrieval — RFC 7515 §4.1.2 makes this a MUST.
- **Validate `kid` against the JWKS** before use. It selects from a known set; it does not specify a path.
- **Never send cookies, auth headers, or AWS metadata creds on JWKS fetches.** RFC 8725 §3.10 explicitly calls this out: "matching the URL to a whitelist of allowed locations and ensuring no cookies are sent."
- **Reject unknown `jku` / `x5u` outright** unless your protocol genuinely federates trust across issuers — and even then, allow-list the hostnames.

Key rotation, the operational half of the problem:

- Issuer publishes the new public key alongside the old one in the JWK Set.
- New tokens carry the new `kid` (RFC 7517: `"kid"` is for "key matching and selection").
- After all old tokens have expired (`exp` window — see [[JWT SE 04 - Expiration and Revocation]]), the issuer drops the old key.
- Verifiers tolerate multiple keys in the set; per-token selection is by `kid`.

Each JWK should bind `kty`, `use`, and `alg` together, so the verifier doesn't have to trust the token's `alg` ([[JWT SE 02 - Algorithm Confusion]]):

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "2024-q1",
      "n": "...", "e": "AQAB"
    }
  ]
}
```

⚠️ "We support `jku` so issuers can self-publish" is a red flag. Unless you allow-list hostnames *and* validate the certificate chain *and* refuse local/RFC1918/metadata addresses, you have shipped SSRF. RFC 8725 §3.10 is unambiguous about the allow-list requirement.

⚠️ Symmetric keys ([[JWT AL 01 - HMAC]]) cannot live in a public JWKS — `use: "sig"` exposes them. RFC 7517 puts it bluntly: "Private and symmetric keys MUST be protected from disclosure to unintended parties."

⚠️ Don't trust the certificate chain in `x5c` without validating it against a CA you trust — having a cert in the header proves nothing.

Cross-references:

- The headers themselves: [[JWT JO 02 - JWS]] (`kid`, `jku`, `x5u`, `x5c`).
- The JWK / JWK Set format: [[JWT JO 05 - JWK]].
- Algorithm pinning per key: [[JWT SE 02 - Algorithm Confusion]], [[JWT JO 06 - JWA]].
- BCP overview: [[JWT SE 01 - RFC 8725 BCP]].
- OIDC's well-known JWKS endpoint: [[JWT UC 03 - OIDC ID Tokens]].

💡 **Takeaway:** the verifier owns the JWKS URL and the trust set; `kid` is a selector, not a path; `jku` and `x5u` are SSRF unless allow-listed.
