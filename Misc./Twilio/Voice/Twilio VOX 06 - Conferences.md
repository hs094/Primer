# VOX 06 — Conferences

🔑 A `<Conference>` is a named multi-party room. Anyone who `<Dial><Conference>RoomName</Conference></Dial>`s the same name lands in the same room. Twilio handles audio mixing.

## Minimal — two callers join one room

Both legs run this TwiML:

```xml
<Response>
  <Dial>
    <Conference>SupportRoom</Conference>
  </Dial>
</Response>
```

The first to arrive waits on hold music; the second causes the conference to start.

## Useful `<Conference>` attributes

```xml
<Conference muted="false"
            beep="onEnter"
            startConferenceOnEnter="true"
            endConferenceOnExit="false"
            waitUrl="https://app.example.com/twiml/wait"
            waitMethod="GET"
            maxParticipants="10"
            record="record-from-start"
            recordingStatusCallback="/twiml/conf-recording"
            statusCallback="/twiml/conf-status"
            statusCallbackEvent="start end join leave mute hold speaker announcement"
            participantLabel="agent"
            coach="CA-of-call-to-coach"
            jitterBufferSize="medium"
            region="us1">
  RoomName
</Conference>
```

| Attr | Notes |
|---|---|
| `startConferenceOnEnter` | Conference starts when this participant enters (typical: agent only). |
| `endConferenceOnExit` | When this participant leaves, conference ends (kicks everyone). Typical: moderator. |
| `muted` | Join muted. |
| `beep` | `true`/`false`/`onEnter`/`onExit`. |
| `waitUrl` | TwiML or audio while waiting. Default = Twilio hold music. |
| `maxParticipants` | Cap (≤ 250 by default). |
| `record` | `record-from-start` \| `do-not-record`. |
| `coach` | A "whisper coach" pattern — coach can hear all, but only the named `coach` CallSid hears the coach. |
| `participantLabel` | Friendly identifier you can target via REST. |
| `region` | Pin mixer region for latency (`us1`, `ie1`, `au1`, `sg1`, `de1`, `jp1`, `br1`). |

## Conference REST API

`Conference` resource (`CF…`). Endpoint: `/Conferences/{ConferenceSid}.json`. Once the room is live:

```python
conf = client.conferences("CF...").fetch()
print(conf.status)   # in-progress / completed
```

Manage participants:

```python
# Add (call out, then bridge in)
client.conferences("CF...").participants.create(
    from_="+15557654321",
    to="+15558675309",
    early_media=True,
    end_conference_on_exit=False,
    label="customer",
)

# Mute / hold a participant
client.conferences("CF...").participants("CA...").update(muted=True)
client.conferences("CF...").participants("CA...").update(hold=True, hold_url="/twiml/hold")

# Kick
client.conferences("CF...").participants("CA...").delete()
```

## Status callbacks

Subscribe to per-conference events with `statusCallbackEvent="start end join leave mute hold speaker announcement"`. Webhook payload includes `ConferenceSid`, `StatusCallbackEvent`, `Reason`, `CallSid`, `ParticipantLabel`, plus event-specific fields.

`speaker` events (active-speaker detection) fire whenever the dominant talker changes — useful for live captions / "now speaking" UIs.

## Coaching pattern

Trainer joins as `coach` of an existing agent's CallSid:

```xml
<Response>
  <Dial>
    <Conference coach="CAagent...">SupportRoom</Conference>
  </Dial>
</Response>
```

Coach hears agent + customer; only the agent hears the coach.

## Whisper before joining

Each `<Dial><Number url="/whisper">+15...</Number></Dial>` lets you play TwiML *to the dialled callee only* before bridging. Combine with a conference: callee hears "incoming customer call about billing", then joins the room.

## Recording

`record="record-from-start"` produces a `RecordingSid` for the entire conference (mixed). Channels: 1 (mixed). For per-participant tracks, use `Dial record="record-from-answer-dual"` per leg or external recording via Media Streams.

## Pricing

- Each participant is a separate billed call leg.
- Conference adds a per-participant-minute fee on top of the call leg cost.
- Recording / transcription priced as per [[Voice/Twilio VOX 05 - Recording]].

⚠️ Conferences with `endConferenceOnExit=false` everywhere will keep ringing forever if nobody leaves intentionally. Always have *one* moderator with `endConferenceOnExit=true`.
