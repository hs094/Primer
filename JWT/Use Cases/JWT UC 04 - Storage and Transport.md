# UC 04 — Storage and Transport

🔑 **One-line: every browser storage choice trades XSS exposure for CSRF exposure — pick the tradeoff that matches your threat model, never localStorage for long-lived tokens.**

## Pattern

The question: where does the token live in the browser, and how does it ride on each request?

Three storage options, three transport patterns:

| Storage | Transport | XSS exposure | CSRF exposure | Cross-site auto-send |
|---|---|---|---|---|
| `localStorage` / `sessionStorage` | JS reads, sets `Authorization: Bearer ...` | **HIGH** — any script reads it | None — not auto-attached | No |
| Memory (JS variable) | JS sets `Authorization: Bearer ...` | Medium — gone on reload, but live JS sees it | None | No |
| Cookie (`HttpOnly`, `Secure`, `SameSite`) | Browser auto-attaches | **None** — JS can't read | **HIGH** unless `SameSite` + CSRF token | Yes (controlled by `SameSite`) |

[RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750) gives the wire format; [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725) hardens the JWT contents. Storage is on you.

## Token shape

The wire is identical regardless of storage. What changes is the request headers.

**Authorization header (localStorage / memory):**

```http
GET /api/me HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

**Cookie:**

```http
GET /api/me HTTP/1.1
Host: api.example.com
Cookie: __Host-session=eyJhbGciOiJSUzI1NiIs...
X-CSRF-Token: 7a3f...
```

A cookie set correctly looks like:

```http
Set-Cookie: __Host-session=eyJhbGciOiJSUzI1NiIs...;
            Path=/;
            Secure;
            HttpOnly;
            SameSite=Lax;
            Max-Age=900
```

Inside, the JWT is unchanged from [[JWT UC 01 - API Authentication]]:

```json
{
  "iss": "https://api.example.com",
  "sub": "user_8f2a91c3",
  "aud": "https://api.example.com",
  "exp": 1714233600,
  "iat": 1714230000,
  "jti": "01HW8XK3VQ5R9N7M2P4T6Y8Z0A"
}
```

## Validation: server-side requirements per transport

Both transports still require everything from [[JWT FN 05 - Validating vs Verifying]] — `iss`, `aud`, `exp`, signature, allowlisted `alg`. The transport decision adds:

**Authorization header:**
- Enforce HTTPS at the edge. RFC 6750 §5.3: "Clients MUST always use TLS (https) or equivalent transport security."
- Set CORS deliberately. Headers don't ride cross-origin without explicit `Access-Control-Allow-Headers: Authorization`.

**Cookie:**
- `Secure` — never sent over plain HTTP. RFC 6750 §5.2: "Bearer tokens MUST NOT be stored in cookies that can be sent in the clear."
- `HttpOnly` — JS cannot read. Defeats token theft via XSS.
- `SameSite=Lax` (default-good) or `Strict` (paranoid) or `None; Secure` (cross-site needed). `Lax` blocks CSRF on POSTs from third-party origins.
- `__Host-` prefix — forces `Secure`, `Path=/`, no `Domain` attribute. Makes subdomain takeover attacks moot.
- Pair with a CSRF defence: double-submit cookie, synchronizer token, or `Origin`/`Referer` check.

## Security tradeoffs (the actual decision)

### localStorage — almost always wrong

Pros: simple, JS can attach to any header, no CSRF concerns.

Cons:
- **Any** XSS = full token theft. Not just "while the page is open" — XSS reads `localStorage.getItem('token')` and exfiltrates.
- Persists across tabs, reloads, days.
- jwt.io intro: "should not store sensitive session data in browser storage due to lack of security."

When to consider: short-lived (≤5 min) tokens with a refresh flow that's also XSS-protected. Even then, prefer memory.

### Memory only

Pros: gone on reload, can't be read after page nav, no CSRF.

Cons: lost on reload — needs silent re-auth (refresh token cookie, OIDC silent renew via iframe, or token endpoint cookie). Still readable by live JS during XSS, but the window is small.

This is the SPA-friendly answer when paired with a refresh-token cookie.

### `HttpOnly` cookie

Pros: XSS cannot exfiltrate it. Browser handles attachment. Combine with `__Host-` and `SameSite=Strict` and you've eliminated most browser-side attacks at once.

Cons:
- CSRF — the browser auto-attaches the cookie on cross-origin requests unless `SameSite` blocks them. Mitigation: `SameSite=Lax` (default for new cookies) plus a CSRF token for state-changing requests.
- Cross-domain APIs: cookies are bound to a domain, awkward for multi-domain APIs.
- Mobile / non-browser clients: less natural than `Authorization: Bearer ...`.

This is the right default for first-party SPAs talking to first-party APIs.

## RFC 8725 (BCP 225) — JWT-content hardening

[RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725) doesn't dictate storage but provides the rules that keep the *contents* safe regardless of where they sit:

- **Algorithm allowlist.** "Libraries MUST enable the caller to specify a supported set of algorithms and MUST NOT use any other algorithms when performing cryptographic operations." See [[JWT SE 05 - Validation Pitfalls]].
- **Validate every cryptographic operation.** Signature, decryption, all of it. Reject the entire token on any failure.
- **Use audience.** "If the same issuer can issue JWTs that are intended for use by more than one relying party or application, the JWT MUST contain an `aud` (audience) claim."
- **Explicit typing.** Use `typ` (e.g. `at+jwt` per RFC 9068, see [[JWT UC 02 - OAuth Access Tokens]]) and check it on consumption — see [[JWT SE 06 - Cross-JWT Confusion]].
- **Mutually exclusive validation rules** for different JWT kinds — distinct `typ`, `aud`, issuers, or keys ([[JWT SE 03 - Key Management]]).
- **Sanitize claims.** §2.9: "Various JWT claims are used by the recipient to perform lookup operations, such as database and LDAP searches" — treat them as untrusted input even after signature verification.
- **TLS in transit.** "If a JWT is cryptographically protected end-to-end by a transport layer, such as TLS using cryptographically current algorithms, there may be no need to apply another layer of cryptographic protections to the JWT." Always TLS in browser contexts.

## ⚠️ Footguns

- **localStorage for long-lived auth.** XSS = total compromise, persistent across sessions. Don't.
- **Cookies without `Secure`.** RFC 6750 §5.2 explicitly: "Bearer tokens MUST NOT be stored in cookies that can be sent in the clear." Browser will happily send the cookie over plain HTTP if you forget `Secure`.
- **Forgetting `HttpOnly`.** Cookie-stored token readable from `document.cookie` is just localStorage with extra steps.
- **`SameSite=None` without `Secure`.** Modern browsers reject this combination — but accidentally setting `SameSite=None` for everything (because cross-origin "didn't work") drops the headline CSRF defense.
- **No CSRF defence with cookies.** `SameSite=Lax` covers most cases but doesn't help when an attacker gets a same-site origin (e.g. user-content subdomain). Pair cookies with double-submit CSRF tokens or origin checks for state-changing requests.
- **Tokens in URL query strings.** RFC 6750 §2.3 SHOULD NOT — they leak into logs, browser history, `Referer` headers sent to third parties. Same for fragment-encoded tokens being later moved by JS into URLs.
- **Mixed transports.** Accepting the same JWT both via cookie and `Authorization` header on the same endpoint multiplies the attack surface. Pick one path per endpoint.
- **CDN / proxy logging.** Even with `Authorization` headers, request logs at intermediaries may capture the header. Configure log-redaction.
- **Token in subresource URLs.** `<img src="https://api/avatar?token=...">` leaks via `Referer` to third parties when the avatar links elsewhere. Just don't.
- **Broken Access-Control headers.** `Access-Control-Allow-Origin: *` with `Allow-Credentials: true` is not allowed but easy to misconfigure. Pin allowed origins.

## Decision matrix

| Scenario | Storage | Transport |
|---|---|---|
| First-party SPA + first-party API | `__Host-` cookie + CSRF token | Cookie |
| SPA + cross-domain API | Memory + refresh-token cookie | `Authorization: Bearer` |
| Mobile app | OS keychain / secure storage | `Authorization: Bearer` |
| Server-to-server | Per-process memory | `Authorization: Bearer` |
| OIDC ID token (RP server) | RP's session store, server-side | Never sent back out |

For the OIDC case see [[JWT UC 03 - OIDC ID Tokens]] — ID tokens stay on the RP server, not in the browser.

## Related

- [[JWT UC 01 - API Authentication]] — header-based transport details
- [[JWT UC 02 - OAuth Access Tokens]] — when the token is an OAuth access token
- [[JWT UC 03 - OIDC ID Tokens]] — different consumer, different storage
- [[JWT SE 04 - Expiration and Revocation]] — short lifetimes + refresh make storage choice less catastrophic
- [[JWT SE 05 - Validation Pitfalls]] — server-side checks that must accompany any transport

💡 **Takeaway:** for browser auth, prefer an `HttpOnly`, `Secure`, `SameSite=Lax` `__Host-` cookie with a CSRF token, fall back to short-lived in-memory tokens with a refresh-cookie, and never use `localStorage` for anything you'd be sad to lose to a stored XSS.
