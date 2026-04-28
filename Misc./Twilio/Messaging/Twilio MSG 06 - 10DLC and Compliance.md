# MSG 06 — 10DLC and Compliance

🔑 To send SMS to US recipients over 10-digit long codes (the cheap default), the carriers require you to register your business as a **Brand** and your use case as a **Campaign**. Unregistered traffic gets aggressively filtered.

## US A2P 10DLC

A2P = Application-to-Person. The flow:

```
Brand (your company)  ──→  Campaign (use case)  ──→  Messaging Service  ──→  Long Code(s)
   identity-verified         vetted by TCR              attach numbers       physical senders
```

**TCR** = The Campaign Registry. Twilio submits to TCR on your behalf.

### Brand types

| Type | Throughput | Notes |
|---|---|---|
| **Standard** | Highest. Trust Score (1–10) drives MPS. | Public/listed companies; full identity vetting. |
| **Low Volume Standard** | Lower MPS, lower fees | < ~6,000 messages/day |
| **Sole Proprietor** | Lowest MPS | Individuals; one campaign limit |

### Campaign use cases

Examples Twilio + TCR will recognise: `2FA`, `Account Notifications`, `Customer Care`, `Delivery Notifications`, `Fraud Alert Messaging`, `Higher Education`, `Marketing`, `Polling and Voting`, `Public Service Announcement`, `Security Alert`. Pick the *narrowest* match — broader categories cost more and face heavier filtering.

### Submitting

Console → Messaging → Regulatory Compliance → A2P 10DLC. Or Brand Registration / Campaign API:

```python
brand = client.messaging.v1.brand_registrations.create(
    customer_profile_bundle_sid="BU...",     # Trust Hub identity
    a2p_profile_bundle_sid="BU...",
    brand_type="STANDARD",
)

# Wait for vetting (minutes → days). Then create a campaign on your Messaging Service.
client.messaging.v1.services("MG...").us_app_to_person.create(
    brand_registration_sid=brand.sid,
    description="Order delivery notifications for example.com customers.",
    message_samples=[
        "Order #1234 shipped — track at https://x.com/t/abcd",
        "Reply STOP to opt out, HELP for help.",
    ],
    us_app_to_person_usecase="DELIVERY_NOTIFICATION",
    has_embedded_links=True,
    has_embedded_phone=False,
)
```

⚠️ Sample messages should look like real production sends. TCR rejects vague placeholders.

### Throughput vs Trust Score

10DLC throughput in messages-per-second (MPS) per number scales with the Brand Trust Score (1–100 normalised; 100=highest):

| Trust Score band | MPS / number (approx) |
|---|---|
| Sole Prop | ~0.3 |
| Low Trust Standard | ~3 |
| Mid Trust Standard | ~30 |
| High Trust Standard | ~75–225 |

(Exact values vary per carrier; treat as planning guidance.)

## Toll-Free Verification (US/CA)

If you use toll-free (`+1888…`, `+1844…`) for SMS to US/CA, submit a **Toll-Free Verification** in Console — sample messages, opt-in flow, business identity. Without verification:

- Throughput is throttled and most messages are filtered.
- Once verified: ~3 MPS per toll-free, no per-campaign fees.

## Short codes

Vanity sender (5–6 digits) — high throughput, brand-owned. Application takes weeks, costs hundreds/month, but no 10DLC registration overhead.

## STOP / HELP / opt-in

Carrier rules (apply to *all* US SMS, registered or not):

- Honour STOP within 5 seconds. Twilio does this for you on Messaging Services.
- Reply to HELP with company name + how to get help + how to stop.
- For marketing campaigns: explicit prior express written consent (TCPA) — store proof.

## Geo Permissions

To send internationally, enable destination countries in Console → Messaging → Geo Permissions. Sending to a disabled country returns `21408`. Helps prevent SMS-pumping fraud (paid traffic to high-cost countries).

## SMS Pumping Protection

Available on Messaging Services and Verify. Detects bot-driven OTP spam to high-risk numbers and blocks before send. Free on Verify; opt-in on Messaging.

## EU / UK

No 10DLC equivalent yet. Use **Mobile** numbers (country-of-recipient mobile prefixes) or **Alphanumeric Sender IDs** (write `from_="MyBrand"` — register the alpha ID per country). One-way only; replies are not possible to alpha senders.

## India

Strict regulatory regime — DLT registration via local carriers, sender ID + template registration. Twilio supports it but onboarding is multi-week.

💡 Default planning answer for "can I just send SMS to country X today?": probably yes via Twilio US fallback, but you'll pay more and face deliverability issues. Register locally for production volumes.
