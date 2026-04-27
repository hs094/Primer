# JO 01 — JOSE Overview

🔑 **JWT is not a single spec — it's the application of the four JOSE specs (JWS, JWE, JWK, JWA) to a JSON claims set.**

## What JOSE is

JOSE = **JSON Object Signing and Encryption**, a family of IETF standards for representing signed and encrypted content using JSON and base64url. It exists because XML-DSig / XML-Enc were heavy and developers wanted compact, URL-safe tokens.

> "JWS represents content secured with digital signatures or Message Authentication Codes (MACs) using JSON-based data structures." — RFC 7515 §1

The four core RFCs ship together (May 2015):

| RFC | Spec | What it does |
|---|---|---|
| **7515** | [[JWT JO 02 - JWS]] | Signs/MACs a payload (integrity + authenticity) |
| **7516** | [[JWT JO 04 - JWE]] | Encrypts a payload (confidentiality + integrity) |
| **7517** | [[JWT JO 05 - JWK]] | Represents a key as a JSON object (+ JWK Sets) |
| **7518** | [[JWT JO 06 - JWA]] | Algorithm registry (HS*, RS*, ES*, AES, ECDH-ES, …) |
| 7519 | [[JWT FN 01 - What Is a JWT]] | JWT = a claims JSON wrapped in JWS or JWE |
| 7520 | — | Cookbook of worked examples |

## How JWT relates

A **JWT is a payload** (a JSON object of [[JWT FN 03 - Registered Claims]]) that is **wrapped** in either:

- **JWS** → signed JWT (the common case, three-part `header.payload.sig`)
- **JWE** → encrypted JWT (five-part, opaque payload)

```
                       ┌──────────────────────┐
   {iss, sub, exp, …}  │   JWS or JWE wrap    │  →  eyJhbGciOi...xxx.yyy.zzz
   (the JWT claims)    └──────────────────────┘
                              ↑
                 algorithms (JWA) + keys (JWK)
```

So "a JWT" without context almost always means a **JWS-signed JWT** — see [[JWT FN 02 - Anatomy and Encoding]] for the wire shape.

## The JOSE Header

Every JWS/JWE starts with a base64url-encoded JSON **JOSE Header**. Shared parameter names across the family:

| Param | Purpose | Defined in |
|---|---|---|
| `alg` | Algorithm (e.g. `HS256`, `RS256`, `RSA-OAEP`) | JWS, JWE |
| `typ` | Media type (`JWT`, `at+jwt`, `JOSE`) | JWS, JWE |
| `cty` | Content type (for nested tokens) | JWS, JWE |
| `kid` | Key ID — selects which JWK to use | JWS, JWE, JWK |
| `jku` / `jwk` | URL of / inline JWK Set | JWS, JWE |
| `x5u` / `x5c` / `x5t` / `x5t#S256` | X.509 cert pointer / chain / thumbprint | JWS, JWE |
| `crit` | Extension params that MUST be understood | JWS, JWE |
| `enc`, `zip` | Content encryption alg, compression (JWE-only) | JWE |

## Serializations

JWS and JWE each define **Compact** (URL-safe, single recipient/sig) and **JSON** (General + Flattened) serializations. JWT itself is **always Compact** — see [[JWT JO 03 - JWS Serializations]].

```
Compact JWS:  BASE64URL(header) . BASE64URL(payload) . BASE64URL(sig)
Compact JWE:  BASE64URL(header) . BASE64URL(enc_key) . BASE64URL(iv) . BASE64URL(ct) . BASE64URL(tag)
```

⚠️ **Footguns at the family level:**
- **Signing ≠ encryption.** A JWS payload is base64url-encoded plaintext anyone can read. Use JWE if you need confidentiality.
- **`alg: none`** is in the registry — and is the source of catastrophic bypasses. See [[JWT AL 05 - none Algorithm]] and [[JWT SE 02 - Algorithm Confusion]].
- **`jku` / `jwk` / `x5u` / `kid` are attacker-controlled** if you trust them blindly. See [[JWT SE 03 - Key Management]].
- **JWT spec only mandates Compact.** Don't expect JSON Serialization in OAuth flows.

## Why this matters in practice

- OAuth / OIDC ID tokens, access tokens, logout tokens: JWS-signed JWTs.
- API keys / session tokens: JWS-signed JWTs (or opaque, if you don't need stateless verify).
- WebAuthn attestation, COSE, FIDO: cousins of JOSE for binary contexts.
- Verify with [[JWT PR 01 - PyJWT]], debug with [[JWT PR 04 - jwt.io Debugger]].

💡 **Takeaway:** JOSE is four small specs that compose. "JWT" is just RFC 7519 saying *"take a JSON claims set, sign it with JWS or encrypt it with JWE, using algorithms from JWA and keys from JWK."*
