# FN 02 — Anatomy and Encoding

🔑 **`header.payload.signature` — three base64url-encoded segments joined by literal dots. The signature covers the first two encoded segments, not the JSON.**

## The Three Parts

```text
xxxxx.yyyyy.zzzzz
  |     |      |
  |     |      +-- signature  (binary, base64url)
  |     +--------- payload    (JSON claims, base64url)
  +--------------- header     (JOSE header, base64url)
```

From RFC 7519 §3:

> A JWT is represented as a sequence of URL-safe parts separated by period ('.') characters. Each part contains a base64url-encoded value.

This is the **JWS Compact Serialization** form — see [[JWT JO 02 - JWS]].

## 1. Header (JOSE Header)

Tells the verifier *how* the token is signed and *what* kind of payload to expect.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg` — signing algorithm. See [[JWT JO 06 - JWA]].
- `typ` — media type, conventionally `"JWT"`.
- `kid` (optional) — key id, used for key rotation against a [[JWT JO 05 - JWK]] set.

## 2. Payload (Claims Set)

A JSON object of claims — see [[JWT FN 03 - Registered Claims]] and [[JWT FN 04 - Public and Private Claims]].

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

⚠️ **The payload is not encrypted.** Base64url is encoding, not encryption. Anyone holding the token can read it.

## 3. Signature

```text
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

Two facts about that input string:

- It signs the **already-encoded** header and payload, joined by a dot.
- Re-encoding the JSON yourself will not reproduce the signature — whitespace, key order, and trailing newlines all matter. Always sign and verify the *exact* bytes you received.

## Base64url, Not Base64

`base64url` is the URL-safe variant defined for JOSE:

- `+` → `-`
- `/` → `_`
- Trailing `=` padding is **stripped**.

So a JWT is safe in URLs, headers, and form bodies without further escaping.

⚠️ **Don't confuse base64 and base64url.** Standard base64 with `+`, `/`, `=` will silently break verification on most libraries. Use the library's JWT helpers, not generic base64 utilities.

## Worked Example (RFC 7519 §3.1)

Header JSON:

```json
{"typ":"JWT",
 "alg":"HS256"}
```

Claims JSON:

```json
{"iss":"joe",
 "exp":1300819380,
 "http://example.com/is_root":true}
```

Compact serialization:

```text
eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ.dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Notice the two dots and the third segment (the signature) is itself base64url.

## Decoding by Hand

```python
import base64, json

token = "eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJIUzI1NiJ9..."
header_b64, payload_b64, sig_b64 = token.split(".")

def b64url_decode(s: str) -> bytes:
    pad = "=" * (-len(s) % 4)
    return base64.urlsafe_b64decode(s + pad)

header = json.loads(b64url_decode(header_b64))
payload = json.loads(b64url_decode(payload_b64))
```

Use this to *inspect*, never to *trust*. Trust requires verification — see [[JWT FN 05 - Validating vs Verifying]].

## Unsecured JWT (RFC 7519 §6)

A JWT with `"alg": "none"` has an **empty** signature segment:

```text
eyJhbGciOiJub25lIn0.eyJpc3MiOiJqb2UiLCJleHAiOjEzMDA4MTkzODAsImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ.
```

Note the trailing dot with nothing after it. Real systems must reject these — see [[JWT AL 05 - none Algorithm]] and [[JWT SE 02 - Algorithm Confusion]].

⚠️ **Never accept `alg: none` in production.** Pin the expected algorithm before verification.

## Encoding Rules Summary

| Step | Input | Output |
|------|-------|--------|
| 1 | JSON header | UTF-8 bytes |
| 2 | UTF-8 bytes | base64url (no padding) |
| 3 | JSON payload | UTF-8 bytes |
| 4 | UTF-8 bytes | base64url (no padding) |
| 5 | `b64h + "." + b64p` | signature bytes |
| 6 | signature bytes | base64url (no padding) |
| 7 | join with dots | `xxxxx.yyyyy.zzzzz` |

💡 **Takeaway:** Three base64url segments, signed over their string concatenation. Encoding is not encryption. Always sign/verify the exact received bytes.
