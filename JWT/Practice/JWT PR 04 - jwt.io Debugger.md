# PR 04 — jwt.io Debugger

🔑 **One-line: the canonical web debugger for inspecting JWTs — paste a token, see the header, payload, and verify the signature live.**

Free, browser-based, hosted by Auth0. The first place you go when a [[JWT FN 01 - What Is a JWT]] won't validate. See companion library docs: [[JWT PR 01 - PyJWT]], [[JWT PR 02 - joserfc Python]], [[JWT PR 03 - jsonwebtoken Node]].

## What it does

- **Decode**: paste a `header.payload.signature` token, see the three [[JWT FN 02 - Anatomy and Encoding]] segments rendered as JSON.
- **Verify**: enter the secret (HS) or public key (RS/ES/EdDSA) and the page tells you whether the signature checks out.
- **Sign**: edit the JSON and the encoded token regenerates live — useful for crafting test fixtures.
- **Generate**: one-click sample token in the algorithm of your choice.

## Supported algorithms

The debugger handles every algorithm in [[JWT JO 06 - JWA]]:

| Family | Algorithms | Note |
|---|---|---|
| HMAC | HS256, HS384, HS512 | [[JWT AL 01 - HMAC]] |
| RSA | RS256, RS384, RS512 | [[JWT AL 02 - RSA Signatures]] |
| RSA-PSS | PS256, PS384, PS512 | |
| ECDSA | ES256, ES256K, ES384, ES512 | [[JWT AL 03 - ECDSA]] |
| EdDSA | Ed25519 | [[JWT AL 04 - EdDSA]] |

For HS* you can toggle "secret base64 encoded" — important when your real secret is a base64-encoded random key (the bytes get decoded before HMAC).

## How to use it

1. Paste the token in the **Encoded** column.
2. The **Decoded** column shows `HEADER`, `PAYLOAD`, and the verification result.
3. For HMAC: paste the secret in the bottom box; signature turns green/red.
4. For RSA / ECDSA / EdDSA: paste the **public key** (PEM) to verify, or both **public + private** to re-sign on edits.

A quick decode roundtrip:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTYyMzkwMjJ9
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Decodes to:

```json
{ "alg": "HS256", "typ": "JWT" }
{ "sub": "1234567890", "name": "John", "iat": 1516239022 }
```

With secret `your-256-bit-secret`, the signature verifies.

You can edit the JSON in the Decoded column and the Encoded column updates — handy for:

- Crafting tokens with custom claims to test your verifier ([[JWT FN 03 - Registered Claims]]).
- Reproducing edge cases (missing `exp`, weird `aud` shapes).
- Showing teammates what each segment of a token actually contains.

## Library catalog (`jwt.io/libraries`)

A curated index of JWT implementations across languages, with a comparison matrix of:

- Sign / verify support
- Standard claim validation: `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`, `typ`
- Algorithm support (HS / RS / PS / ES / EdDSA)
- GitHub stars

Languages covered include: .NET, C, C++, Clojure, Crystal, D, Dart, Delphi, Deno, Elixir, Erlang, Go, Haskell, Java, JavaScript, Kotlin, Lua, Node, Objective-C, OCaml, Perl, PHP, Python, Ruby, Rust, Scala, Swift, TypeScript and more.

Notable entries: `golang-jwt` (~6.5k stars), `auth0/node-jsonwebtoken` (~18k stars, see [[JWT PR 03 - jsonwebtoken Node]]), `pyjwt` (see [[JWT PR 01 - PyJWT]]).

## Browser extension

Auth0 also ships a Chrome extension ("JWT Debugger") that decodes any JWT visible on the page or in the clipboard — handy when debugging frontend localStorage tokens or `Authorization: Bearer ...` headers in DevTools.

## Common patterns

- **Confirm a `kid` matches your JWKS.** Decode the header, copy `kid`, grep your `/.well-known/jwks.json`.
- **Eyeball expiration.** The decoded payload shows `exp` as a Unix timestamp; jwt.io renders it as a human date inline.
- **Catch encoding bugs.** If a server uses base64 (with padding) instead of base64url, jwt.io will refuse to decode — that's the bug, not jwt.io.
- **Sanity-check `alg`.** If your token says `alg: none` or `alg: HS256` when you expected `RS256`, you've hit [[JWT SE 02 - Algorithm Confusion]] territory.

## Footguns

⚠️ **Don't paste production tokens into a third-party site.** jwt.io claims everything runs client-side in JavaScript and tokens never leave your browser — but a real bearer token grants access to whatever it grants access to. Use a local debugger or generate a synthetic token with the same shape. See [[JWT SE 05 - Validation Pitfalls]].

⚠️ **Don't paste production secrets either.** Same reason. For HS256 verification in the debugger, use a *test* secret with a *test* token.

⚠️ **The debugger's "verified" badge is not a security audit.** It only confirms the signature math. It does NOT check `exp`, `aud`, `iss`, `nbf`, or any business-logic claims. Validation ≠ verification ([[JWT FN 05 - Validating vs Verifying]]).

⚠️ **Some libraries listed in the catalog are unmaintained.** GitHub stars ≠ active maintenance. Cross-check the last commit date and CVE history before adopting (notable example: `python-jose` — replaced by [[JWT PR 02 - joserfc Python]]).

⚠️ **`alg: none`** *can* be encoded by the debugger. That's a real RFC 7519 algorithm — and the reason every production verifier should reject it explicitly. See [[JWT SE 03 - Key Management]].

💡 **Takeaway:** jwt.io is the fastest way to inspect a token and sanity-check signing — but treat tokens like passwords (don't paste prod), and remember "signature verified" is a long way from "claims valid."
