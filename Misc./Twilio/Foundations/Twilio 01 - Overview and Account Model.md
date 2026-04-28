# 01 — Overview and Account Model

🔑 Twilio = a set of REST APIs around carrier networks (PSTN, SS7, WhatsApp, RCS, email, video). You pay per use; you build the app.

## What's in the platform

- **Communications**: Messaging (SMS / MMS / RCS / WhatsApp), Voice, Video, Email (SendGrid), Fax (legacy).
- **Identity & risk**: Verify (OTP / push / TOTP / SNA), Lookup (line type, SIM swap, fraud).
- **Builder tools**: Studio (visual flows), Functions / Assets (Node runtime), TwiML Bins (static TwiML).
- **Data**: Segment (CDP), Event Streams (Kinesis-style fan-out of platform events).
- **Telecom plumbing**: Phone Numbers, Elastic SIP Trunking, Trust Hub (brand / 10DLC / toll-free).

## Account model

```
Organization (enterprise)
└── Account (primary)        ← AccountSid: AC...
    ├── Subaccount A         ← AccountSid: AC...   (own credentials, isolated billing/usage)
    ├── Subaccount B
    └── Phone Numbers, Apps, Messaging Services, …
```

- **AccountSid** — `AC[32 hex]`. Identifies the account; appears in every API path.
- **AuthToken** — symmetric secret paired with the AccountSid. Used to sign outbound calls *and* webhooks.
- **Subaccounts** — child accounts under one parent; common pattern for multi-tenant SaaS to isolate per-customer billing and credentials.
- **Twilio Connect** — auth-on-behalf-of model so a third-party app can act inside another customer's Twilio account without you holding their token.

## SID prefixes (worth memorising)

| Prefix | Resource |
|---|---|
| `AC` | Account |
| `SK` | API Key |
| `PN` | Phone Number (`IncomingPhoneNumber`) |
| `MG` | Messaging Service |
| `SM` | SMS Message |
| `MM` | MMS Message |
| `CA` | Call |
| `RE` | Recording |
| `CF` | Conference |
| `VA` | Verify Service |
| `VE` | Verification |
| `FW` | Studio Flow |
| `FN` | Studio Execution |
| `IS` | Conversations Service |
| `CH` | Conversation |
| `RM` | Video Room |
| `ZS` | Functions Service |

💡 Treat the two-letter prefix as the *type* — every Twilio "thing" is a SID.

## Regions and residency

- Default control plane: **US1** (Virginia). Other edges: **IE1** (Ireland), **AU1** (Sydney), **JP1** (Tokyo), **BR1** (São Paulo), **SG1** (Singapore), **DE1** (Frankfurt).
- Helper libraries take a `region` / `edge` to pin requests: `Client(sid, token, region="ie1", edge="dublin")`.
- ⚠️ Region only changes *where the API request lands*. Carrier delivery still goes wherever the destination number lives.

## Console vs API

- Anything in the Console (UI at `console.twilio.com`) maps to a REST resource. If you can click it, you can `POST` it.
- Prefer Studio for non-developer-edited flows; prefer code (Functions or your own service) for anything you need to test, version-control, or unit-test.
