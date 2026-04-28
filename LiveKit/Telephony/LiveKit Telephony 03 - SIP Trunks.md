# 03 — SIP Trunks

🔑 Trunks are the bridge between your SIP provider and LiveKit. Inbound trunks accept calls; outbound trunks place them. Two separate objects, both authenticated by username/password (Cloud has no static IPs).

Source: https://docs.livekit.io/sip/trunk-outbound/ and https://docs.livekit.io/sip/trunk-inbound/

## Inbound trunk
Accepts INVITEs from your provider on configured numbers.

```json
{
  "name": "My inbound trunk",
  "numbers": ["+15551234567"],
  "allowed_addresses": [],
  "allowed_numbers": [],
  "auth_username": "user",
  "auth_password": "pass",
  "krisp_enabled": true
}
```

Key fields:
- `numbers` — DIDs this trunk accepts INVITEs for.
- `allowed_addresses` — IP/CIDR allowlist for the provider (skip on Cloud, prefer auth).
- `allowed_numbers` — caller-ID allowlist; empty = any number (then auth is required).
- `krisp_enabled` — server-side Krisp noise cancellation.

API:
```python
from livekit import api
lkapi = api.LiveKitAPI()
trunk = await lkapi.sip.create_sip_inbound_trunk(
    api.CreateSIPInboundTrunkRequest(trunk=api.SIPInboundTrunkInfo(...))
)
```

Note: Twilio Elastic SIP Trunking does **not** support inbound username/password auth — use IP allowlist on non-Cloud or a different provider.

## Outbound trunk
Places INVITEs to a SIP provider's address.

```json
{
  "name": "My outbound trunk",
  "address": "sip.telnyx.com",
  "numbers": ["+15551234567"],
  "auth_username": "user",
  "auth_password": "pass",
  "transport": "SIP_TRANSPORT_AUTO",
  "destination_country": "US"
}
```

Key fields:
- `address` — provider host (e.g. `sip.telnyx.com`, `mytrunk.pstn.twilio.com`).
- `numbers` — caller-ID number(s); set to `["*"]` to allow any caller-ID specified at call time via `sip_number`.
- `destination_country` — region-pin egress for compliance/quality.
- `transport` — UDP/TCP/TLS/AUTO.

API:
```python
trunk = await lkapi.sip.create_sip_outbound_trunk(
    api.CreateSIPOutboundTrunkRequest(trunk=api.SIPOutboundTrunkInfo(...))
)
```

## Management ops (both directions)
- `ListSIPInboundTrunk` / `ListSIPOutboundTrunk`
- `UpdateSIPInboundTrunkFields` — patch specific fields, keep ID.
- `UpdateSIPInboundTrunk` — full replace, keep ID.
- `DeleteSIPTrunk`

## Tags
[[LiveKit]] [[Telephony]] [[SIP]] [[Trunks]]
