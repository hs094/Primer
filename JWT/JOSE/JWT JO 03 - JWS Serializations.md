# JO 03 — JWS Serializations

🔑 **One signed object, three on-the-wire shapes: Compact (URL-safe, single sig), JSON General (multi-sig), JSON Flattened (single sig, JSON-friendly). JWT only uses Compact.**

## Why three forms

A JWS is logically *(header, payload, signature)*. RFC 7515 (§3, §7) gives you three encodings depending on what you optimize for:

| Form | Sigs | URL-safe | Use when |
|---|---|---|---|
| **Compact** | 1 | yes | HTTP `Authorization` headers, query strings, cookies — i.e. **JWTs** |
| **JSON General** | many | no | One payload, multiple signers (e.g. notarization, escrow) |
| **JSON Flattened** | 1 | no | API bodies / storage where JSON is more ergonomic than dot-form |

> "JWS objects with multiple signatures are not supported by the JWS Compact Serialization." — RFC 7515 §7

## Compact Serialization (§3.1, §7.1)

Three base64url segments, two dots, no whitespace. **Only the Protected Header**, no unprotected header.

```
BASE64URL(UTF8(JWS Protected Header)) || '.' ||
BASE64URL(JWS Payload)               || '.' ||
BASE64URL(JWS Signature)
```

Concrete:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Properties:

- ASCII only, URL-safe, no padding.
- Single signature, single header.
- This is the only form that JWT (RFC 7519) actually mandates.

## JSON General Serialization (§3.2, §7.2)

A JSON object with a `signatures` array. Each entry can have its own protected header, unprotected header, and signature — all over the same `payload`.

```json
{
  "payload": "eyJzdWIiOiIxMjM0NTY3ODkwIn0",
  "signatures": [
    {
      "protected": "eyJhbGciOiJSUzI1NiJ9",
      "header":    { "kid": "key-1" },
      "signature": "cC4hiUPo..."
    },
    {
      "protected": "eyJhbGciOiJFUzI1NiJ9",
      "header":    { "kid": "key-2" },
      "signature": "DtEhU3lj..."
    }
  ]
}
```

- `payload` is base64url-encoded once and shared.
- `protected` is the base64url JSON Protected Header (signed).
- `header` is the **Unprotected Header** (NOT signed).
- Each entry is verified independently; the JWS is valid if at least one sig verifies and the app accepts that algorithm.

## JSON Flattened Serialization (§3.2, §7.2.2)

Optimization for the single-signer case — pulls one entry up to the top level, drops the array.

```json
{
  "payload":   "eyJzdWIiOiIxMjM0NTY3ODkwIn0",
  "protected": "eyJhbGciOiJSUzI1NiJ9",
  "header":    { "kid": "key-1" },
  "signature": "cC4hiUPo..."
}
```

Strict equivalence: a Flattened JWS with one signature carries the same information as a General JWS with a single-element `signatures` array.

## Signing input is identical across forms

> "Both serializations produce identical signature values for identical parameters." — paraphrased from RFC 7515 §7

The bytes you sign over are always:

```
ASCII( BASE64URL(UTF8(Protected Header)) || '.' || BASE64URL(Payload) )
```

Unprotected headers are **never** part of the signing input — they exist for routing hints only.

## Comparison at a glance

| Aspect | Compact | JSON General | JSON Flattened |
|---|---|---|---|
| Sigs | 1 | N | 1 |
| Unprotected hdr | no | yes | yes |
| Wire form | dot-separated b64url | JSON object with `signatures[]` | JSON object, no array |
| URL-safe | yes | no (must encode) | no |
| Used by JWT | **yes** | no | no |
| Used by JOSE-Cookbook examples | yes | yes | yes |

⚠️ **Footguns:**
- **Don't put security-relevant params in Unprotected Header** — they're not signed. `kid` is borderline acceptable as a hint; `alg` is not.
- **A JWS from JSON General with one trusted sig and one untrusted sig is still "valid" by the per-signature rule.** Your code must check **which** signature verified and whether that signer's algorithm/key is acceptable.
- **Flattened ≠ Compact.** Flattened is still JSON; Compact is the dot form. They're not interchangeable on the wire.
- **JWT (RFC 7519) only specifies Compact.** Don't ship a JSON-serialization JWT and expect OIDC libraries to consume it.
- **Whitespace-sensitive comparisons fail.** Verifiers MUST recompute over the **received** base64url bytes, not over a re-canonicalized header.

See:
- [[JWT JO 02 - JWS]] for header parameters and verification flow
- [[JWT FN 02 - Anatomy and Encoding]] for the JWT-specific Compact wire format

💡 **Takeaway:** Compact is the JWT form (one signer, header + payload + sig, dotted, URL-safe). JSON General/Flattened exist for multi-signer or JSON-native pipelines — same signing input, different envelope.
