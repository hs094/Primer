# 10 — Magic Links and Passwordless

🔑 Email a single-use link with a signed token; clicking it logs the user in. No password to forget, no password to phish — but the email inbox becomes the security boundary.

## The Flow
1. User enters email → server generates random token (≥ 128 bits), stores hash + `expires_at` + `email`.
2. Server emails `https://app/auth/verify?token=<token>`.
3. User clicks → server looks up hash, checks expiry, marks consumed, mints session.

## Token Hygiene
| Property | Why |
|---|---|
| **Cryptographically random** | `secrets.token_urlsafe(32)` — never `uuid4()` or timestamps |
| **Hashed at rest** | Store `sha256(token)`, not the raw token (DB leak ≠ account takeover) |
| **Single use** | Mark consumed atomically; reject reuse |
| **Short TTL** | 5–15 minutes; not 24 hours |
| **Bound to email** | Don't accept the token if the user already changed their email |
| **Binding to device/IP (optional)** | Reduces same-link-different-browser attacks |

## Variants
| Variant | Tradeoff |
|---|---|
| **Magic link** | One-tap; but URL leaks if the email forwards |
| **Email OTP code** | User retypes 6–8 digits; survives email forwarding |
| **Same-device link + cross-device code** | Best UX; show code on origin device, link in email |

## Threat Model
⚠️ **The link is the credential.** Anyone with read access to the inbox is the user. Implications:
- Corporate email systems, archive bots, browser extensions, "preview link" scanners (Outlook, Slack unfurl) may *prefetch* the link → consumes the token before the user clicks.
- Mitigations: require POST to consume (not GET), use a landing page that the *user* must click again, or a OTP code instead.
- Phishing: an attacker triggers the link to *their* email by registering with the victim's email — useless without inbox access, but rate-limit anyway.

## Rate Limits & Abuse
- Per-email: max N requests / window.
- Per-IP: cap signups.
- Captcha after threshold.
- Don't reveal "no such email" — same response for new vs existing user (avoid enumeration).

💡 Magic links shine for low-frequency apps (newsletters, B2B tools). For daily-use apps, pair with passkeys ([[Auth 11 - MFA and Passkeys]]) so users don't need email round-trips. See [[Auth 02 - JWT vs Sessions]] for what to mint after.

## Tags
[[Auth]] [[Magic Link]] [[Passwordless]] [[Email OTP]] [[Tokens]]
