# AL 05 — `none` Algorithm

🔑 **One-line: `alg: none` means the JWS has no signature — historically the cause of catastrophic auth bypasses, and disabled by default in any well-behaved library.**

## How it works

RFC 7518 §3.6 defines an "Unsecured JWS" — a JWS with the `alg` header set to `none` and an **empty** signature segment.

> *"JWSs MAY also be created that do not provide integrity protection. Such a JWS is called an Unsecured JWS."* — RFC 7518 §3.6

> *"An Unsecured JWS uses the 'alg' value 'none' and is formatted identically to other JWSs, but MUST use the empty octet sequence as its JWS Signature value."* — RFC 7518 §3.6

The token still has three dot-separated segments — the third is just empty:

```
eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9.
```

Decoded header: `{"alg":"none"}`. Decoded payload: `{"sub":"admin"}`. No signature. Anyone can write that string. There is **nothing** preventing modification.

## In a JWT header

```json
{ "alg": "none" }
```

(`typ: JWT` is conventional but not required — and there is no `kid`, because there is no key.)

## Why it exists at all

It's intended for cases where integrity is provided by a *different* layer — e.g. a JWT embedded inside an already-signed envelope, or one carried over a mutually-authenticated TLS channel where signing again adds no security. RFC 7518 is explicit:

> *"Implementations that support Unsecured JWSs MUST NOT accept such objects as valid unless the application specifies that it is acceptable for a specific object to not be integrity protected."* — RFC 7518 §3.6

> *"Implementations MUST NOT accept Unsecured JWSs by default. In order to mitigate downgrade attacks, applications MUST NOT signal acceptance of Unsecured JWSs at a global level, and SHOULD signal acceptance on a per-object basis."* — RFC 7518 §3.6

## RFC 8725 (BCP 225) reinforces this

The JWT Best Current Practices RFC (see [[JWT SE 01 - RFC 8725 BCP]]) gives §3.1 to algorithm verification:

> *"Libraries MUST enable the caller to specify a supported set of algorithms and MUST NOT use any other algorithms when performing cryptographic operations."* — RFC 8725 §3.1

> *"each key MUST be used with exactly one algorithm, and this MUST be checked when the cryptographic operation is performed."* — RFC 8725 §3.1

And specifically on `none`:

> *"JWT libraries SHOULD NOT generate JWTs using 'none' unless explicitly requested … SHOULD NOT consume JWTs using 'none' unless explicitly requested by the caller."* — RFC 8725

## Signing with PyJWT (don't, but for completeness)

PyJWT requires you to opt in twice — algorithm string `"none"` *and* key `None`:

```python
import jwt

# Encoding "alg: none" — you must pass key=None and algorithm="none"
token = jwt.encode({"sub": "demo"}, key=None, algorithm="none")
# 'eyJhbGciOiJub25lIn0.eyJzdWIiOiJkZW1vIn0.'

# Decoding requires explicit opt-in via algorithms=["none"]
claims = jwt.decode(token, key=None, algorithms=["none"], options={"verify_signature": False})
```

You must pass `options={"verify_signature": False}` *and* include `"none"` in `algorithms`. Even then, what you've decoded is **untrusted attacker-controlled JSON.**

## Key requirements

There is no key. That is the entire problem.

## The classic attack: alg-confusion-to-none

Around 2014–2018, dozens of JWT libraries (Node `jsonwebtoken` < 4.2.2, Java `auth0/java-jwt` < 3.4, PHP `firebase/php-jwt` < 5.0, etc.) had a verify path that looked like this:

```python
# DANGEROUS — pseudocode of the historical bug
def verify(token, key):
    header = decode_header(token)
    alg = header["alg"]                  # attacker-controlled
    if alg == "none":
        return decode_payload(token)     # accepts unsigned token
    return verify_signature(token, key, alg)
```

Attacker takes any valid JWT, replaces `alg` with `none`, drops the signature segment, and the server treats the forged payload as authentic. **Pure auth bypass, no key knowledge required.** This is why pinning `algorithms=` at every verify call is non-negotiable.

⚠️ **Never put `"none"` in your `algorithms=` list** when verifying production tokens. Not even "just for testing." Tests have a way of becoming production.

⚠️ **Never auto-select algorithm from the token header.** Always pass `algorithms=["RS256"]` (or whatever you actually use). See [[JWT SE 02 - Algorithm Confusion]].

⚠️ **Audit your dependency tree.** Old versions of every major JWT library had the alg-confusion-to-none bug. Pin minimum versions:
  - PyJWT ≥ 2.0
  - `jsonwebtoken` (Node) ≥ 9.0
  - `firebase/php-jwt` ≥ 6.0
  - `auth0/java-jwt` ≥ 4.0

⚠️ **Capitalization tricks.** Some buggy verifiers compared `alg` case-insensitively or normalized it; attackers used `None`, `NONE`, `nOnE` to slip past blocklists. The fix is allowlist (`algorithms=["RS256"]`), not blocklist.

⚠️ **`alg: none` ≠ unencoded payload.** Don't confuse the unsecured JWS (`alg: none`) with the unencoded payload option from RFC 7797 (`b64: false`). Both surfaces have been used in confusion attacks; neither belongs in your verifier without an explicit, documented reason.

⚠️ **CVE pattern.** Search "JWT alg none CVE" — there are still new ones every year, usually in less-maintained libraries or custom verify code.

## When `none` is acceptable

Almost never. The only legitimate uses are:

- A JWT used as a structured payload format inside an already-authenticated channel (e.g. mTLS, signed envelope).
- Local-only debugging where no trust boundary exists.

In every other case, **a JWT with `alg: none` is just a JSON object with extra steps.** Use plain JSON or sign it.

## Related

- [[JWT FN 01 - What Is a JWT]]
- [[JWT FN 02 - Anatomy and Encoding]]
- [[JWT JO 02 - JWS]]
- [[JWT JO 06 - JWA]]
- [[JWT SE 01 - RFC 8725 BCP]]
- [[JWT SE 02 - Algorithm Confusion]]
- [[JWT SE 03 - Key Management]]
- [[JWT PR 01 - PyJWT]]
- [[JWT PR 03 - jsonwebtoken Node]]
- [[JWT AL 01 - HMAC]]
- [[JWT AL 02 - RSA Signatures]]
- [[JWT AL 03 - ECDSA]]
- [[JWT AL 04 - EdDSA]]

💡 **Takeaway:** `alg: none` is a footgun by design — pin `algorithms=` to a positive allowlist, never include `"none"`, and treat any token claiming `alg: none` as forged.
