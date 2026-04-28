# VFY 01 — Verify API

🔑 Verify is a turnkey OTP/MFA backend. You don't store the code, you don't generate it, you don't pick the channel — you call `Verifications.create(...)` and `VerificationCheck.create(...)` and Verify handles delivery, expiry, retries, fraud heuristics, and SMS pumping protection.

## Resources

```
Service (VA…)                 ← per-app config: rate limits, branding, channels
├── Verification (VE…)        ← one OTP attempt
├── VerificationCheck         ← validate a user-supplied code
├── RateLimit                 ← configurable per-service (per-IP, per-phone, etc.)
├── Entity                    ← user object (TOTP / Push factor owner)
│   ├── Factor                ← TOTP secret or push device key
│   └── Challenge             ← per-factor verification request
└── AccessToken               ← short-lived JWT for SDK auth (TOTP / push)
```

## Channels

| Channel | Notes |
|---|---|
| `sms` | Default. Subject to 10DLC / toll-free verification in US. |
| `call` | TTS reads the code aloud — accessibility / fallback when SMS filtered. |
| `email` | Requires SendGrid integration. |
| `whatsapp` | Cheaper than SMS in many markets. Requires WABA. |
| `sna` (Silent Network Auth) | Carrier-API based; user opens a link, carrier confirms ownership without OTP. Mobile-data only. |
| `totp` | Authenticator app (RFC 6238). Per-user `Factor` + `Challenge`. |
| `push` | Native iOS/Android Verify SDK with biometric approval. |

## Hello-world flow

```python
service_sid = "VA..."

# 1. Send OTP
client.verify.v2.services(service_sid).verifications.create(
    to="+15551234567",
    channel="sms",
)

# 2. User submits code (e.g. via your form)
check = client.verify.v2.services(service_sid).verification_checks.create(
    to="+15551234567",
    code="123456",
)

if check.status == "approved":
    # log the user in
    ...
```

`status` on `VerificationCheck`: `pending`, `approved`, `canceled`, `max_attempts_reached`, `expired`.

## Service configuration

```python
service = client.verify.v2.services.create(
    friendly_name="ACME Login",
    code_length=6,
    lookup_enabled=True,                    # check Lookup before sending (block VoIP / risky lines)
    sms_pumping_enabled=True,               # Fraud Guard
    psd2_enabled=False,                     # adds a transaction-binding hash to OTPs (EU PSD2)
    skip_sms_to_landlines=True,
    dtmf_input_required=False,              # voice channel only
    do_not_share_warning_enabled=True,      # adds anti-phishing footer
)
```

## Rate limits

Per-service, optional. Each is a named bucket:

```python
limit = client.verify.v2.services(svc).rate_limits.create(
    unique_name="per_phone_5_per_hour", description="5 sends/hour per phone",
)
client.verify.v2.services(svc).rate_limits(limit.sid).buckets.create(max=5, interval=3600)
```

Then pass the bucket key when creating a verification:

```python
client.verify.v2.services(svc).verifications.create(
    to="+15551234567",
    channel="sms",
    rate_limits={"per_phone_5_per_hour": "+15551234567"},
)
```

## Custom code (deterministic OTP)

For testing or pre-registered codes:

```python
client.verify.v2.services(svc).verifications.create(
    to="+15551234567",
    channel="sms",
    custom_code="000000",
)
```

⚠️ Disable in production via Console; static codes are a credential.

## Custom message / template

Defaults are localised. Override per-language by registering a template:

```python
client.verify.v2.templates.list()           # built-ins
# Or set verification.template_sid + template_custom_substitutions
```

## TOTP (authenticator app)

```python
# 1. Create entity (user) and TOTP factor
entity = client.verify.v2.services(svc).entities.create(identity="user-42")
factor = client.verify.v2.services(svc).entities("user-42").new_factors.create(
    friendly_name="ACME Login",
    factor_type="totp",
)
# factor.binding contains an otpauth:// URI you render as a QR for the user

# 2. User enters first code from their app — verify
client.verify.v2.services(svc).entities("user-42").factors(factor.sid).update(
    auth_payload="123456",
)

# 3. Subsequent logins — challenge against the factor
client.verify.v2.services(svc).entities("user-42").challenges.create(
    factor_sid=factor.sid, auth_payload="654321",
)
```

## Push (Verify SDK)

Native iOS / Android / RN SDK with biometric prompt. Server flow: create `Entity` → register `Factor` (`push`) → create `Challenge` (mobile receives push, user approves with FaceID) → check `Challenge.status == approved`.

## Silent Network Auth (SNA)

User opens a magic link on mobile data; carrier confirms phone-number ownership without an OTP UI.

```python
v = client.verify.v2.services(svc).verifications.create(
    to="+15551234567",
    channel="sna",
)
print(v.sna.url)   # send to user; redirect them through it
```

⚠️ Mobile-data only (Wi-Fi off). US/UK/AU/SE/DE/IT/ES + select carriers; check the latest support matrix.

## Fraud Guard

Free with Verify on SMS. Detects SMS-pumping campaigns (bots requesting OTPs to premium-rate countries) before send. Set `sms_pumping_enabled=True` on the service. Blocked attempts return `60410` and don't bill.

## Verify Attempts API

For retros / dashboards:

```python
attempts = client.verify.v2.verification_attempts.list(
    date_created_after=datetime(2026, 4, 1, tzinfo=timezone.utc),
    channel_data={"to": "+15551234567"},
)
for a in attempts:
    print(a.channel, a.conversion_status, a.price)
```

## Why Verify over rolling-your-own SMS OTP

- Built-in rate limiting + Fraud Guard.
- Carrier deliverability per-region (Verify uses optimised routes).
- Auto-fallback channels (SMS → voice).
- No code / hash / expiry storage on your side.
- 10DLC compliance handled (Verify uses dedicated routes; you don't need a campaign for it).
