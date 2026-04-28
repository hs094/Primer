# CV 01 — Conversations

🔑 The Conversations API is Twilio's **two-way, multi-channel chat** abstraction. One `Conversation` can have participants on SMS, MMS, WhatsApp, and chat-SDK identities (web/mobile users) all in the same thread, with persisted message history.

## Why use Conversations vs raw Messages

| | `Messages` API | `Conversations` API |
|---|---|---|
| State | stateless — each send is independent | persistent thread per conversation |
| Channels | one-channel-per-send | mixes SMS / MMS / WhatsApp / chat in one thread |
| In-app chat | n/a (you'd build it) | first-class via Conversations Chat SDK |
| Message history | query by phone number, manually correlate | server-side ordered, paginated |
| Webhooks | per-resource (status callback) | per-conversation, plus global service-level |

Pick `Conversations` for support inboxes, chatbots that may move from SMS to in-app, or any case where you want a unified "thread" object.

## Resource shape

```
Service (IS…)                  ← optional, multi-tenant container
└── Conversation (CH…)
    ├── Participant (MB…)      ← one per channel address (phone / chat identity)
    ├── Message (IM…)          ← author identity, body, media, attributes
    ├── Webhook                ← scoped to the conversation
    └── Delivery Receipt       ← one per participant per message
User (US…)                     ← global identity for chat-SDK clients
Role (RL…)                     ← permissions (channel/conversation roles)
Credential (CR…)               ← APNS/FCM creds for push to chat clients
```

## Quickstart — SMS-backed conversation

```python
# Create a conversation
conv = client.conversations.v1.conversations.create(friendly_name="Order #1234")

# Add the customer (SMS participant)
client.conversations.v1.conversations(conv.sid).participants.create(
    messaging_binding_address="+15551234567",            # customer
    messaging_binding_proxy_address="+15557654321",      # your Twilio number
)

# Optionally: add an in-app participant by chat identity
client.conversations.v1.conversations(conv.sid).participants.create(identity="agent_alice")

# Send from your side
client.conversations.v1.conversations(conv.sid).messages.create(
    author="agent_alice",
    body="Hi — how can I help?",
)
```

When the customer replies via SMS, Twilio routes the inbound into the same conversation, fires a webhook, and persists the message. Your in-app SDK client (using identity `agent_alice`) sees it instantly.

## Auto-creating conversations from inbound SMS

On a Messaging Service, set `Inbound Routing` → `Auto-create Conversations`. Now any inbound SMS to a number on that service creates (or appends to) a conversation keyed on `(messaging_binding_address, messaging_binding_proxy_address)`. No glue code.

## WhatsApp participants

```python
client.conversations.v1.conversations(conv.sid).participants.create(
    messaging_binding_address="whatsapp:+15551234567",
    messaging_binding_proxy_address="whatsapp:+14155238886",
)
```

24-hour-window rules from [[Messaging/Twilio MSG 05 - WhatsApp]] still apply.

## Webhooks

Two scopes:

- **Service-level** (catch-all): `conversations.v1.services(svc).configuration.webhooks.update(...)` — fires for every event in every conversation in the service.
- **Conversation-scoped**: attach a webhook to one conversation (e.g. forward only #1234 to a specific URL).

Events worth subscribing to:

| Event | When |
|---|---|
| `onMessageAdded` | New message in any conversation |
| `onMessageUpdated` / `Removed` | Edits / deletes |
| `onConversationAdded` / `Updated` / `Removed` | Lifecycle |
| `onParticipantAdded` / `Removed` | Membership |
| `onDeliveryUpdated` | Delivery receipt status changed |
| `onTypingStarted` / `Ended` | Chat-SDK clients only |

Set `prePostHooks` on the service config to make webhooks **synchronous** (Twilio waits for your 200 before delivering / accepting messages). Useful for moderation; expensive in latency.

## Chat SDK (in-app)

Web JS + iOS + Android SDKs let you treat the same `Conversation` as an in-app chat thread:

```js
import { Client } from "@twilio/conversations";

const client = new Client(accessTokenJwt);
const conv   = await client.getConversationBySid("CH...");
await conv.sendMessage("hello from the web");
conv.on("messageAdded", (m) => render(m));
```

The Access Token is a short-lived JWT minted server-side, scoped to a chat identity (e.g. `agent_alice`). See your service's API key + signing config.

## Message attributes

Free-form JSON metadata per message — useful for tagging which assistant generated a reply, or for storing rich-card payload references:

```python
client.conversations.v1.conversations(conv).messages.create(
    author="bot",
    body="Here are 3 options",
    attributes='{"intent":"options","model":"gpt-4o"}',
)
```

Read back via `message.attributes` (string JSON).

## Delivery receipts

Each outbound message produces one `DeliveryReceipt` per participant with `status` ∈ `queued`, `sent`, `delivered`, `read`, `failed`, `undelivered`. Subscribe to `onDeliveryUpdated` or fetch on demand:

```python
for r in client.conversations.v1.conversations(conv).messages("IM...").delivery_receipts.list():
    print(r.participant_sid, r.status)
```

## Closing / archiving

Set `state` on a conversation: `active` → `inactive` (read-only) → `closed` (read-only, hidden by default). No deletion required for compliance — `closed` keeps the audit trail.

## Compared to Flex

Flex (Twilio's contact-center product) is built on top of Conversations + Voice + TaskRouter. If you only need the chat thread + multi-channel routing, Conversations alone is enough; Flex adds agent UI + skill-based routing.
