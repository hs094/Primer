# 11 — MFA and Passkeys

🔑 MFA = "something you know + something you have / are." Passkeys collapse both into one resident credential the device guards with biometric — phishing-resistant by design, the long-term replacement for passwords.

## MFA Factor Types
| Factor | Examples | Phishing-resistant? |
|---|---|---|
| Knowledge | Password, PIN, security question | No |
| Possession | TOTP app, hardware key, push, SMS | Mostly no (TOTP/push phishable, hardware yes) |
| Inherence | Fingerprint, FaceID, voice | Local — combined with possession via passkeys |

## TOTP (RFC 6238)
- HMAC-SHA1 of (shared secret, current 30-sec time window) → 6-digit code.
- Setup: server shows `otpauth://` URI as QR; Authy/1Password/Google Authenticator stores the secret.
- Verify with ±1 step skew tolerance for clock drift.
- ⚠️ Phishable: a fake login page can relay the code in real time.

## Push & SMS
- **SMS**: cheap, universally compatible; SIM-swap attacks make it the weakest factor. NIST deprecates for high-assurance.
- **Push** (Auth0 Guardian, Duo): better UX, vulnerable to "MFA fatigue" spamming. Number-matching variants help.

## WebAuthn / Passkeys (FIDO2)
🔑 The only widely-deployed phishing-resistant factor. Browser/OS holds a key pair per origin; signing requires user verification (biometric/PIN). No shared secret crosses the wire.

| Concept | Detail |
|---|---|
| **Credential** | (public key, credential id) bound to `rp.id` (your domain) |
| **User verification** | Biometric/PIN unlocks the private key locally |
| **Resident key** | Stored on the authenticator → usernameless flow |
| **Attestation** | Optional proof of authenticator model (mostly enterprise) |
| **Sync vs device-bound** | iCloud Keychain, Google Password Manager sync passkeys across devices; security keys (YubiKey) stay device-bound |

### Why "phishing-resistant"
The browser passes the *origin* to the authenticator; signature is bound to it. A phish on `app.evil.com` cannot get a signature valid for `app.com`. The user can't be tricked into "typing the code into the wrong site" because there's no code.

## Recovery Codes
- One-time codes generated at MFA enrollment.
- Hashed at rest, single-use, regenerable.
- Without them, a lost authenticator → support ticket → social-engineering vector.

## Practical Stack
- **Primary**: passkey (or password + passkey upgrade prompt).
- **Fallback**: TOTP.
- **Last resort**: recovery codes.
- Avoid SMS unless your users have no other option.

💡 All major providers ship this: [[Auth 05 - Clerk]], [[Auth 08 - Auth0]], [[Auth 06 - Supabase Auth]], [[Auth 07 - Better Auth]] all support passkeys. See [[WebAuthn]].

## Tags
[[Auth]] [[MFA]] [[Passkeys]] [[WebAuthn]] [[TOTP]] [[FIDO2]]
