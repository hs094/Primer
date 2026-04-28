# LKP 01 — Lookup v2

🔑 Lookup answers questions about a phone number *before you send*. Use it to: normalise to E.164, classify line type (mobile/VoIP/landline), check fraud risk, detect SIM swaps, validate ownership. **v2** is the current API; v1 is deprecated.

## Endpoint

```
GET /v2/PhoneNumbers/{PhoneNumber}?Fields={comma-sep-fields}&CountryCode=US
```

`PhoneNumber` is whatever the user typed (e.g., `(415) 999 9999`); `CountryCode` lets Lookup parse local format. Without `Fields`, you get just basic format info (E.164, national format, country code) — that part is **free**.

## Data packages (`Fields`)

| Field | What you get | Typical price |
|---|---|---|
| `validation` | E.164 format + country | free |
| `line_type_intelligence` | mobile / fixedVoip / nonFixedVoip / landline / tollFree / personal / pager | $ |
| `caller_name` | CNAM (US/CA only) | $ |
| `sim_swap` | recent SIM-swap activity flag + date | $$ |
| `call_forwarding` | whether unconditional call forwarding is active | $$ |
| `line_status` | active / inactive / unknown (handset reachability) | $$ |
| `identity_match` | name / address / DOB match score against user-supplied claim | $$ |
| `reassigned_number` | risk that this number was reassigned since a given date (TCPA defence) | $$ |
| `sms_pumping_risk` | risk score 0–100 that this number is part of an SMS-pumping campaign | $ |
| `phone_number_quality_score` | 0–100 trust/reach score | $ |
| `pre_fill` | name + address + DOB suggestions for KYC pre-fill | $$ |

Ask for multiple in one request:

```python
phone = client.lookups.v2.phone_numbers("(415) 999-9999").fetch(
    fields="line_type_intelligence,sms_pumping_risk",
    country_code="US",
)

print(phone.phone_number)                           # +14159999999 (E.164)
print(phone.line_type_intelligence["type"])        # mobile
print(phone.sms_pumping_risk["risk_score"])        # 0..100
```

## Common patterns

### Normalise user input before storing

```python
def to_e164(raw: str, default_country: str = "US") -> str:
    p = client.lookups.v2.phone_numbers(raw).fetch(country_code=default_country)
    if not p.valid:
        raise ValueError(f"invalid: {p.validation_errors}")
    return p.phone_number
```

`valid` is `False` if Lookup can't parse it; `validation_errors` is a list (e.g. `["TOO_SHORT"]`).

### Block VoIP / landlines for SMS

```python
info = client.lookups.v2.phone_numbers(num).fetch(fields="line_type_intelligence")
if info.line_type_intelligence["type"] in {"landline", "fixedVoip", "nonFixedVoip"}:
    raise ValueError("SMS not supported for this number")
```

### Pre-OTP fraud check

```python
score = client.lookups.v2.phone_numbers(num).fetch(fields="sms_pumping_risk").sms_pumping_risk
if score["risk_score"] >= 75:
    block_otp(num, reason="high pumping risk")
```

(Or just enable Verify Fraud Guard, which uses the same signal automatically.)

### TCPA reassigned-number defence

```python
ra = client.lookups.v2.phone_numbers(num).fetch(
    fields="reassigned_number",
    last_verified_date="2024-01-15",      # last time you confirmed consent
).reassigned_number

if ra["last_verified_date"] != "2024-01-15" or ra["is_number_reassigned"] == "yes":
    revoke_consent(num)
```

The `reassigned_number` package returns `is_number_reassigned: yes|no|unknown`.

### SIM swap (anti-fraud at login)

```python
ss = client.lookups.v2.phone_numbers(num).fetch(fields="sim_swap").sim_swap
# ss["last_sim_swap"]["last_sim_swap_date"] in past 24/72/720h ⇒ require step-up auth
```

### Identity match (KYC)

```python
m = client.lookups.v2.phone_numbers(num).fetch(
    fields="identity_match",
    first_name="Pranav", last_name="S",
    address_line_1="…", postal_code="…", country_code="US",
).identity_match
# m["first_name_match"], m["last_name_match"], m["address_match"], m["summary_score"]
```

## Pricing model

- Validation = free.
- Each priced field = separate per-lookup charge.
- Errors (number invalid) don't bill the priced fields.
- Bundle fields in one request — no discount but one fewer round-trip.

## When to use Lookup vs Verify vs Trust Hub

- **Lookup** = answer questions about a number on demand.
- **Verify** = send / check OTPs (uses Lookup internally if `lookup_enabled`).
- **Trust Hub** = persistent compliance / brand registration (10DLC, CNAM).

⚠️ Lookup is **synchronous** and adds 100–600 ms. For high-throughput sign-up paths, cache results per number for hours/days where the underlying fact is stable (line type, country) and only re-pull risk/SIM-swap fields fresh.
