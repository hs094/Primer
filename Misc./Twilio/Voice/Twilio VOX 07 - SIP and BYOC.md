# VOX 07 — SIP and BYOC

🔑 Twilio bridges the PSTN and your existing SIP / VoIP gear in two directions: **SIP termination** (pstn → SIP), **SIP origination** (SIP → pstn), and **BYOC** (Bring Your Own Carrier) so Twilio routes calls *over your* upstream carrier instead of buying minutes from Twilio.

## The three SIP products

| Product | What it does |
|---|---|
| **SIP Domain** (`Sip.Domains`) | A Twilio-hosted SIP endpoint (`xyz.sip.us1.twilio.com`) that runs TwiML on inbound. Use to receive SIP calls from softphones / PBX. |
| **Elastic SIP Trunking** | Twilio sells you DIDs + SIP origination/termination for your on-prem PBX. No TwiML — pure SIP carrier. |
| **BYOC Trunks** | Programmable Voice routes the *carrier leg* over a third-party carrier you've signed up with directly. You keep using TwiML / Calls API; Twilio just hands the leg to your trunk. |

## SIP Domain — receive SIP into TwiML

```python
domain = client.sip.domains.create(
    domain_name="example.sip.us1.twilio.com",
    voice_url="https://app.example.com/twiml/voice",
    voice_method="POST",
)

# Authentication: register a credential list and attach it
creds = client.sip.credential_lists.create(friendly_name="agents")
client.sip.credential_lists(creds.sid).credentials.create(username="alice", password="strong-pw")
client.sip.domains(domain.sid).auth.calls.credential_list_mappings.create(credential_list_sid=creds.sid)
```

Now `sip:alice@example.sip.us1.twilio.com` (auth: alice/strong-pw) hits your `voice_url` like a regular inbound call. The webhook payload includes `From=sip:alice@…` and `Direction=inbound`.

### IP ACL alternative

Instead of (or alongside) credentials, allowlist source IPs:

```python
acl = client.sip.ip_access_control_lists.create(friendly_name="office")
client.sip.ip_access_control_lists(acl.sid).ip_addresses.create(
    friendly_name="office-1", ip_address="203.0.113.42",
)
```

## Make outbound to a SIP URI

```python
client.calls.create(
    to="sip:user@example.com:5060;transport=tls",
    from_="+15557654321",
    url="https://app.example.com/twiml/connected",
    sip_auth_username="me",
    sip_auth_password="...",
)
```

Or in TwiML:

```xml
<Dial>
  <Sip username="me" password="...">sip:user@example.com:5060;transport=tls</Sip>
</Dial>
```

`transport=tls` enforces SIP-TLS. Add `srtp=true` URI parameter to require SRTP for media.

## Elastic SIP Trunking

Use case: you have an on-prem (or cloud) PBX (Asterisk, FreeSWITCH, 3CX, RingCentral, Cisco CUCM) and want Twilio as the PSTN carrier.

```python
trunk = client.trunking.v1.trunks.create(
    friendly_name="HQ trunk",
    domain_name="hq.pstn.twilio.com",
)

# Origination = inbound to your PBX
client.trunking.v1.trunks(trunk.sid).origination_urls.create(
    friendly_name="primary",
    sip_url="sip:pbx-1.example.com:5060",
    weight=10, priority=10, enabled=True,
)

# Termination = outbound from your PBX (auth)
client.trunking.v1.trunks(trunk.sid).credentials_lists.create(credential_list_sid=creds.sid)

# Assign Twilio numbers to the trunk
client.trunking.v1.trunks(trunk.sid).phone_numbers.create(phone_number_sid="PN...")
```

No TwiML, no `Calls` API in the path. Calls flow as raw SIP between the PSTN and your PBX. Twilio Console → Voice → Manage → Trunks shows registrations + active calls.

## BYOC — keep Twilio Programmable Voice, route via your carrier

```python
trunk = client.voice.v1.byoc_trunks.create(
    friendly_name="ACME carrier",
    voice_url="https://app.example.com/twiml/answer",
    voice_method="POST",
    from_domain_sid="...",
    connection_policy_sid="NY...",
)
```

Then create calls with `byoc=trunk.sid` (REST) — outbound legs route via the BYOC trunk to your upstream carrier. Inbound from your carrier hits the BYOC trunk's `voice_url` like any Twilio inbound. Useful if you have wholesale rates with another carrier and just want the programmable layer.

## Codec / SRTP / TLS

- Default codecs: `PCMU`, `PCMA` (G.711). Add `OPUS` via Console.
- TLS: `sip:user@host;transport=tls`. Highly recommended.
- SRTP: include `srtp=true` URI parameter. Required for some compliance regimes.

## Common SIP errors

| SIP code | Meaning |
|---|---|
| 401 / 407 | Auth challenge — credentials wrong or missing. |
| 403 | IP not in ACL. |
| 404 | Number not assigned to your trunk. |
| 480 | User busy / temporarily unavailable. |
| 486 | Busy here. |
| 488 | Codec mismatch. |
| 503 | Trunk overloaded / no upstream available. |
| 603 | Decline. |

⚠️ A 488 in production almost always means OPUS not enabled or transport= mismatch. Pin codec list explicitly in your PBX.

## Voice Insights — diagnosing SIP issues

Voice Insights surfaces per-call MOS, jitter, packet loss, RTT for both call legs. Required reading for SIP integrations: most "calls drop after 30s" issues are NAT pinholes or RTP timeouts that only Insights makes visible.
