# 01 — Overview

🔑 LiveKit SIP bridges PSTN to realtime rooms, exposing callers as standard `Participant` objects so AI agents can answer and place calls.

Source: https://docs.livekit.io/sip/

## What it is
LiveKit Telephony lets you build AI-powered voice apps that handle inbound and outbound calls. Callers become standard LiveKit participants in a room — no special code path for telephony participants once they're in.

## Three core primitives
- **SIP Participant**: a caller/callee inside a Room. Same APIs as any participant (publish, subscribe, attributes, metadata).
- **Trunk**: bridges a third-party SIP provider (Twilio, Telnyx, Plivo, …) or a LiveKit Phone Number. Two flavors:
  - `SIPInboundTrunk` — accepts incoming calls.
  - `SIPOutboundTrunk` — places outgoing calls.
- **Dispatch Rule**: routes inbound callers into rooms, sets participant attributes/metadata, can dispatch agents.

## Architecture
Three components cooperate:
1. **DID source** — phone number (LiveKit Phone Numbers or 3rd-party).
2. **LiveKit server** — manages rooms, participants, API.
3. **LiveKit SIP** — terminates SIP, applies dispatch rules.

```
PSTN ──▶ SIP Provider ──▶ LiveKit SIP ──▶ Dispatch Rule ──▶ Room ──▶ Agent
```

## Supported SIP features
- Transports: UDP, TCP, TLS
- Media: RTP, SRTP, HD voice (Opus), DTMF (RFC 2833 `telephone-event/8000`)
- Transfers: cold + warm (SIP REFER)
- Caller ID, SIP OPTIONS keepalive
- Krisp noise cancellation, region pinning, secure trunking

## Limitations
- No video over SIP
- No SIP Registration (REGISTER)
- No SIPREC
- LiveKit Cloud has no static IP range — use username/password, not IP allowlists

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[VoiceAI]]
