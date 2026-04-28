# 05 — Phone Numbers

🔑 A phone number on Twilio is the `IncomingPhoneNumber` resource. It carries: the number itself, capabilities, and the *webhook URLs* Twilio hits when something happens.

## Number types

| Type | Format | Notes |
|---|---|---|
| **Local** | `+1415…` | Cheapest, geographic. Limited SMS throughput. |
| **Toll-Free** | `+1888…` `+1844…` etc. | High SMS throughput in US/CA after **toll-free verification**. |
| **Mobile** | Mobile-prefixed E.164 | Required for SMS in many non-NA countries. |
| **Short Code** | 5–6 digits | High throughput, brand owns it, requires application + carrier approval (weeks). |
| **Alphanumeric Sender ID** | up to 11 chars | One-way SMS only, country-dependent (mostly EU/APAC). |

All numbers are **E.164** — `+`, country code, subscriber number, no spaces or dashes.

## Search and buy

```python
# Search available US local numbers in area code 415 with SMS
available = client.available_phone_numbers("US").local.list(
    area_code=415,
    sms_enabled=True,
    limit=20,
)

# Buy one
number = client.incoming_phone_numbers.create(
    phone_number=available[0].phone_number,
    sms_url="https://app.example.com/twilio/sms",
    voice_url="https://app.example.com/twilio/voice",
)
print(number.sid)   # PN...
```

## Configure webhooks

`IncomingPhoneNumber` URLs:

| Field | Fires when |
|---|---|
| `voice_url` (+ `voice_method`) | Inbound call. Response = TwiML. |
| `voice_fallback_url` | `voice_url` 5xx'd or timed out. |
| `status_callback` | Call lifecycle events. |
| `sms_url` | Inbound SMS. Response = TwiML `<Message>`. |
| `sms_fallback_url` | Same idea for messages. |
| `sms_status_callback` | Inbound message processed. |

Or — set them all at once by attaching the number to a **Messaging Service** (`messaging_service_sid`) or **TwiML Application** (`voice_application_sid`); the service holds the URLs centrally.

```python
client.incoming_phone_numbers(number.sid).update(
    voice_url="https://app.example.com/twilio/voice",
    sms_url="https://app.example.com/twilio/sms",
    status_callback="https://app.example.com/twilio/voice-status",
)
```

## Capabilities

Each number carries a `capabilities` dict: `voice`, `sms`, `mms`, `fax`. Filter at search time — buying a non-SMS number for SMS is a common foot-gun.

## Porting

Move an existing number into Twilio: submit a port-in request via Console (or Porting REST API). Numbers stay live on the previous carrier until cutover; expect 2–6 weeks for North America, longer for international.

## Number pools — Messaging Service vs raw number

- **One number** → simple, single sender. Fine for low-volume.
- **Messaging Service** (`MG…`) → pool many numbers, Twilio picks per-recipient. Required for Sticky Sender, geomatch, A2P 10DLC compliant scaling. See [[Messaging/Twilio MSG 04 - Messaging Services]].

## Trust Hub (compliance)

- **CNAM** — caller-ID name registration (US).
- **SHAKEN/STIR** — caller-ID attestation; required to avoid "Spam Likely" on US carriers.
- **A2P 10DLC** — register your brand + campaigns to send SMS over US 10-digit long codes. See [[Messaging/Twilio MSG 06 - 10DLC and Compliance]].
- **Toll-Free Verification** — required for any production SMS over toll-free in US/CA.

⚠️ Unregistered 10DLC sending is heavily filtered (most messages dropped) since 2023.
