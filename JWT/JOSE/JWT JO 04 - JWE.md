# JO 04 — JWE (JSON Web Encryption)

🔑 **A JWE has *five* base64url parts (not three) because content encryption needs a header, an encrypted CEK, an IV, the ciphertext, and an AEAD auth tag.**

## What JWE is (RFC 7516)

> "JSON Web Encryption (JWE) represents encrypted content using JSON-based data structures." — RFC 7516 §1

JWE provides **confidentiality + integrity** (JWS provides only integrity). Internally it always uses **AEAD** for the actual content encryption, and a separate **key management** step decides how the recipient gets the per-message Content Encryption Key (CEK).

## Compact serialization — the five parts

```
BASE64URL(UTF8(JWE Protected Header))  || '.' ||
BASE64URL(JWE Encrypted Key)           || '.' ||
BASE64URL(JWE Initialization Vector)   || '.' ||
BASE64URL(JWE Ciphertext)              || '.' ||
BASE64URL(JWE Authentication Tag)
```

| # | Part | What it is |
|---|---|---|
| 1 | **Protected Header** | JSON: `alg`, `enc`, `kid`, optional `zip`, `cty`, … |
| 2 | **Encrypted Key** | The CEK wrapped/encrypted per `alg` (empty for `dir` and `ECDH-ES`) |
| 3 | **IV** | Initialization Vector for the AEAD `enc` algorithm |
| 4 | **Ciphertext** | The encrypted payload |
| 5 | **Authentication Tag** | AEAD tag covering the AAD (= encoded protected header) + ciphertext |

The **AAD** (additional authenticated data) is `ASCII(BASE64URL(UTF8(Protected Header)))` — that's why tampering with the header invalidates decryption.

## Two algorithm parameters

JWE splits the work in two:

| Param | Picks | Examples |
|---|---|---|
| `alg` | **Key management** alg — how the CEK is delivered | `RSA-OAEP-256`, `A256KW`, `ECDH-ES`, `ECDH-ES+A128KW`, `dir`, `PBES2-HS256+A128KW` |
| `enc` | **Content encryption** alg (must be AEAD) | `A256GCM`, `A128CBC-HS256`, `A192CBC-HS384`, `A256CBC-HS512` |

The `enc` value is non-optional and **MUST be an AEAD** with a fixed key length.

`zip` is optional and currently only `DEF` (DEFLATE) is registered. It must be in the Protected Header (so it's integrity-covered).

## The five Key Management Modes (§2, §4.6)

| Mode | What happens to the CEK | Example `alg` | Encrypted Key field |
|---|---|---|---|
| **Key Encryption** | CEK encrypted with recipient public key | `RSA-OAEP-256`, `RSA1_5` | non-empty |
| **Key Wrapping** | CEK wrapped with a shared symmetric KEK | `A128KW`, `A256KW`, `A128GCMKW` | non-empty |
| **Direct Key Agreement** | CEK derived directly via ECDH-ES; no wrap | `ECDH-ES` | **empty** |
| **Key Agreement with Key Wrapping** | ECDH-ES → KEK → wrap CEK | `ECDH-ES+A128KW`, `+A256KW` | non-empty |
| **Direct Encryption** | Pre-shared symmetric key IS the CEK | `dir` | **empty** |

For `dir` and `ECDH-ES`, segment 2 is **the empty string** — `header..iv.ct.tag`. Two consecutive dots are correct.

## Concrete shape

```
eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJlbmMiOiJBMjU2R0NNIn0
.
OKOawDo13gRp2ojaHV7LFpZcgV7T6DVZKTyKOMTYUmKoTCVJRgckCL9kiMT03JGe...
.
48V1_ALb6US04U3b
.
5eym8TW_c8SuK0ltJ3rpYIzOeDQz7TALvtu6UG9oMo4vpzs9tX_EFShS8iB7j6jiSdiwkIr3ajwQzaBtQD_A
.
XFBoMYUZodetZdvTiFvSkQ
```

Five segments, four dots — fixed.

## Header parameters (in addition to the JWS set)

| Param | Notes |
|---|---|
| `alg` | **REQUIRED** — key management algorithm |
| `enc` | **REQUIRED** — content encryption AEAD |
| `zip` | Optional compression (`DEF`). MUST be integrity-protected |
| `epk` | Ephemeral public key (for `ECDH-ES*`) |
| `apu` / `apv` | AgreementPartyUInfo / VInfo (for ECDH-ES KDF) |
| `iv`, `tag` | For `A*GCMKW` modes (the KEK-wrap IV/tag, not the content one) |
| `p2s`, `p2c` | PBES2 salt / iteration count |
| `kid`, `jku`, `jwk`, `x5*`, `typ`, `cty`, `crit` | Inherited from JWS — same semantics |

For nested JWS-inside-JWE ("signed-then-encrypted" JWT), set `cty: "JWT"` on the outer JWE.

## Producing a JWE — sequence

1. Pick `alg` and `enc`.
2. Determine the **CEK** (random for wrap/encrypt modes; derived for ECDH-ES; pre-shared for `dir`).
3. Compute the **JWE Encrypted Key** per `alg` (empty for `dir`/`ECDH-ES`).
4. Generate a fresh random **IV** of the size required by `enc`.
5. Build + base64url-encode the Protected Header → that's the **AAD**.
6. AEAD-encrypt the (optionally `zip`-compressed) plaintext under the CEK with the IV and the AAD; receive ciphertext + tag.
7. Concatenate the five base64url parts with `.`.

Validation reverses this; AEAD verifies the tag before returning plaintext, so any tamper in header / iv / ciphertext fails closed.

## JSON serialization

Same logical pieces, broken out into a JSON object so multiple recipients can each have their own `encrypted_key`:

```json
{
  "protected":   "<base64url Protected Header>",
  "unprotected": { "...": "..." },
  "recipients": [
    { "header": { "alg": "RSA-OAEP-256", "kid": "rsa-1" }, "encrypted_key": "..." },
    { "header": { "alg": "ECDH-ES+A128KW", "kid": "ec-1" }, "encrypted_key": "..." }
  ],
  "iv":         "...",
  "ciphertext": "...",
  "tag":        "...",
  "aad":        "..."   
}
```

`aad` lets you bind extra data into the auth tag without putting it in the protected header.

⚠️ **Footguns:**
- **Two `alg` values exist in JWE** — the key-management `alg` and the content-encryption `enc`. Mixing them up is the #1 confusion.
- **`dir` mode publishes the CEK section as empty.** Don't reject empty segments — they're spec-correct for some `alg`s.
- **`A*CBC-HS*` is the *encrypt-then-MAC* AES-CBC mode**, not raw CBC. Never use plain `AES-CBC` with a separate HMAC you wrote yourself.
- **Compression before encryption (`zip: DEF`) leaks payload structure** through ciphertext length — beware CRIME/BREACH-style attacks for attacker-influenced plaintexts.
- **Algorithm allow-list applies here too.** A recipient that auto-accepts any `alg`/`enc` is exploitable (e.g. `RSA1_5` Bleichenbacher). See [[JWT SE 02 - Algorithm Confusion]] and [[JWT SE 03 - Key Management]].
- **JWE protects confidentiality but says nothing about authenticity of the *sender*.** Anyone with the recipient's public key can produce a valid JWE — sign-then-encrypt (nested JWT) when sender identity matters.
- **Don't reuse IVs.** Required to be unique per (key, encryption) pair for GCM; the encryptor MUST generate fresh randomness.

See [[JWT JO 05 - JWK]] for how recipient keys are represented and [[JWT JO 06 - JWA]] for the full algorithm registry.

💡 **Takeaway:** JWE = `header.encrypted_key.iv.ciphertext.tag`. `alg` decides how the CEK gets to the recipient (wrap / encrypt / agree / direct), `enc` is the AEAD that actually encrypts the payload. AEAD makes header tampering automatic-fail.
