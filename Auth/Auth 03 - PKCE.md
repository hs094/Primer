# 03 — PKCE

🔑 PKCE (RFC 7636, "pixie") binds the authorization code to the client that started the flow, so a stolen code is useless without the secret only that client knows. **Required** for public clients (SPA, mobile, native) — recommended for everyone.

Source: https://datatracker.ietf.org/doc/html/rfc7636

## The Problem
Public clients can't safely hold a `client_secret` (anyone can decompile the app, sniff the SPA bundle). Without one, the auth code redirect is the *only* secret — and on mobile it travels via custom URL schemes that another app can register and steal.

## The Solution: A Per-Request Secret
1. Client generates a random **`code_verifier`** (43–128 chars).
2. Computes **`code_challenge = BASE64URL(SHA256(code_verifier))`** with `code_challenge_method=S256`.
3. Sends `code_challenge` on `/authorize`.
4. Auth server stores it against the issued code.
5. On `/token`, client sends the original `code_verifier`.
6. Server hashes it, compares — must match, or token request fails.

```
verifier:  dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
challenge: E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
```

⚠️ **Always use `S256`.** `plain` is in the spec for legacy devices but gives zero protection.

## What It Defends Against
- **Authorization code interception** on mobile (custom URL schemes), in proxies, in browser history.
- An attacker with the code but not the verifier cannot redeem it.

## What It Doesn't Defend Against
- A compromised client where the attacker reads both verifier and code (XSS in your SPA, malware on device). PKCE is for *interception*, not full client compromise.
- Phishing of the auth page itself.

💡 Modern OAuth 2.1 makes PKCE mandatory for *all* auth code flows, public and confidential. Just always do it. See [[Auth 01 - OAuth 2.0 and OIDC]].

## Tags
[[Auth]] [[OAuth]] [[PKCE]] [[SPA]] [[Mobile Auth]]
