# Twilio Knowledge Pack

Crisp, scannable revision notes covering the major Twilio products (twilio.com/docs).
Source: https://www.twilio.com/docs

[[Twilio]] [[automation]] [[phone-calling]]

## How to use
- Each note = one concept, optimized for recall.
- Code blocks are the minimum that exercises the feature (Python where the SDK is shown).
- 🔑 = core idea. ⚠️ = gotcha. 💡 = mental model. 🧪 = testable habit.

## Foundations

| # | Topic | Note |
|---|---|---|
| 01 | Account model, SIDs, regions, console layout | [[Foundations/Twilio 01 - Overview and Account Model]] |
| 02 | Helper libraries, auth (Account SID + Auth Token, API Keys) | [[Foundations/Twilio 02 - Helper Libraries and Auth]] |
| 03 | Webhook request signing — `X-Twilio-Signature`, `RequestValidator` | [[Foundations/Twilio 03 - Webhook Signature Validation]] |
| 04 | Error codes, retries, status callbacks pattern | [[Foundations/Twilio 04 - Errors and Status Callbacks]] |
| 05 | Phone numbers — buy, port, configure, IncomingPhoneNumber | [[Foundations/Twilio 05 - Phone Numbers]] |

## Messaging

| # | Topic | Note |
|---|---|---|
| 01 | Send SMS — `client.messages.create(...)` | [[Messaging/Twilio MSG 01 - Send SMS]] |
| 02 | Receive SMS + reply with TwiML `<Message>` | [[Messaging/Twilio MSG 02 - Receive SMS and TwiML]] |
| 03 | `Message` resource — fields, statuses, lifecycle | [[Messaging/Twilio MSG 03 - Message Resource]] |
| 04 | Messaging Services — sender pool, Sticky Sender, geomatch | [[Messaging/Twilio MSG 04 - Messaging Services]] |
| 05 | WhatsApp Business — sandbox, templates (Content), 24h window | [[Messaging/Twilio MSG 05 - WhatsApp]] |
| 06 | A2P 10DLC, toll-free verification, compliance | [[Messaging/Twilio MSG 06 - 10DLC and Compliance]] |
| 07 | MMS, media URLs, Content API templates | [[Messaging/Twilio MSG 07 - MMS and Content Templates]] |

## Voice

| # | Topic | Note |
|---|---|---|
| 01 | Outbound calls — `client.calls.create(...)` | [[Voice/Twilio VOX 01 - Outbound Calls]] |
| 02 | Inbound calls — webhook + TwiML response | [[Voice/Twilio VOX 02 - Inbound Calls and TwiML]] |
| 03 | TwiML voice reference — verbs and nouns cheatsheet | [[Voice/Twilio VOX 03 - TwiML Voice Reference]] |
| 04 | `Call` resource — fields, statuses, modify in flight | [[Voice/Twilio VOX 04 - Call Resource]] |
| 05 | Recordings — `<Record>`, dual-channel, transcription | [[Voice/Twilio VOX 05 - Recording]] |
| 06 | `<Conference>` — multi-party, moderators, hold | [[Voice/Twilio VOX 06 - Conferences]] |
| 07 | SIP, Elastic SIP Trunking, BYOC | [[Voice/Twilio VOX 07 - SIP and BYOC]] |
| 08 | Media Streams — bidirectional audio over WebSocket | [[Voice/Twilio VOX 08 - Media Streams]] |

## Verify, Lookup, Conversations

| # | Topic | Note |
|---|---|---|
| VFY | Verify API — Service / Verification / VerificationCheck | [[Verify/Twilio VFY 01 - Verify API]] |
| LKP | Lookup v2 — line type, SIM swap, identity match, fraud risk | [[Lookup/Twilio LKP 01 - Lookup v2]] |
| CV  | Conversations — multi-channel chat, participants, services | [[Conversations/Twilio CV 01 - Conversations]] |

## Video, SendGrid, Studio, Serverless

| # | Topic | Note |
|---|---|---|
| VID | Programmable Video — Rooms, Tracks, access tokens | [[Video/Twilio VID 01 - Programmable Video]] |
| SG  | SendGrid — Mail Send v3, dynamic templates, event webhook | [[SendGrid/Twilio SG 01 - SendGrid Mail and Templates]] |
| STU | Studio — Flows, widgets, Executions REST API | [[Studio/Twilio STU 01 - Studio Flows]] |
| SVL | Functions & Assets — Node runtime, Serverless Toolkit | [[Serverless/Twilio SVL 01 - Functions and Assets]] |

## Source map

```
Foundations/    → twilio.com/docs/usage  + helper libraries
Messaging/      → twilio.com/docs/messaging
Voice/          → twilio.com/docs/voice
Verify/         → twilio.com/docs/verify
Lookup/         → twilio.com/docs/lookup
Conversations/  → twilio.com/docs/conversations
Video/          → twilio.com/docs/video
SendGrid/       → twilio.com/docs/sendgrid
Studio/         → twilio.com/docs/studio
Serverless/     → twilio.com/docs/serverless
```
