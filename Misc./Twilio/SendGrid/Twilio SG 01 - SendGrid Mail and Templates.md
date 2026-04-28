# SG 01 — SendGrid Mail and Templates

🔑 SendGrid (a Twilio company) is Twilio's email API. It runs on a separate domain (`api.sendgrid.com`) with its own auth (API keys, not Twilio AuthToken). Use it for transactional + marketing email; webhook-driven event tracking is the headline feature.

## Auth

```python
import os, sendgrid
from sendgrid.helpers.mail import Mail

sg = sendgrid.SendGridAPIClient(os.environ["SENDGRID_API_KEY"])
```

The API key is created in the SendGrid dashboard (or via SSO from Twilio Console). Scope = "Full Access" or restricted (mail send only, marketing only, …).

## Send a basic email

```python
from sendgrid.helpers.mail import Mail

message = Mail(
    from_email="hello@example.com",
    to_emails="user@example.com",
    subject="Welcome to ACME",
    plain_text_content="Thanks for signing up.",
    html_content="<p>Thanks for <b>signing up</b>.</p>",
)
resp = sg.send(message)
print(resp.status_code, resp.headers["X-Message-Id"])
```

Endpoint: `POST https://api.sendgrid.com/v3/mail/send`. 202 = accepted, **not** delivered.

## Personalizations

The native model for "send the same template to many recipients with per-recipient variables":

```json
POST /v3/mail/send
{
  "from": {"email": "hello@example.com"},
  "template_id": "d-abc123…",
  "personalizations": [
    { "to": [{"email": "alice@example.com"}], "dynamic_template_data": {"first_name": "Alice", "code": "X1Y2"} },
    { "to": [{"email": "bob@example.com"}],   "dynamic_template_data": {"first_name": "Bob",   "code": "Q9R7"} }
  ]
}
```

Up to 1000 personalizations per request. Each is one logical email; SendGrid expands template + variables independently.

## Dynamic templates

Authored in SendGrid UI (Design Library) using **Handlebars**. `template_id` = `d-…` (the `d-` prefix marks dynamic vs legacy templates).

Inside the template:

```html
<p>Hi {{first_name}},</p>
{{#if order}}
  <p>Order #{{order.id}} totals {{order.total}}.</p>
{{/if}}
```

Pass any JSON to `dynamic_template_data` — Handlebars resolves `{{path.to.field}}`.

## Attachments

```python
from sendgrid.helpers.mail import Attachment, FileContent, FileName, FileType, Disposition
import base64, pathlib

with pathlib.Path("invoice.pdf").open("rb") as f:
    encoded = base64.b64encode(f.read()).decode()

message.attachment = Attachment(
    FileContent(encoded),
    FileName("invoice.pdf"),
    FileType("application/pdf"),
    Disposition("attachment"),
)
```

⚠️ Total payload (incl. base64 attachments) ≤ 30 MB per request. ≥ 10 MB triggers throttling.

## Categories & custom args

For analytics + segmentation:

```python
message.add_category("welcome-flow")
message.add_custom_arg("user_id", "42")
```

Both surface in the Event Webhook payload — use `custom_args` for opaque IDs, `categories` for grouping.

## Suppressions

Hard-bounces, spam reports, unsubscribes — SendGrid maintains lists per group:

| Suppression group | Why a recipient lands here |
|---|---|
| Global Unsubscribes | Hit any unsubscribe link |
| Group Unsubscribes | Per-group (e.g. marketing only) |
| Bounces | Permanent failures |
| Blocks | ISP blocks |
| Spam Reports | Marked as spam |
| Invalid Emails | Syntax / domain doesn't exist |

You can list/remove via the Suppressions API. Sending to a suppressed address silently drops (no bill).

## Event Webhook

Subscribe a URL in SendGrid → Settings → Mail Settings → Event Webhook. Receives JSON arrays:

```json
[
  { "email": "alice@example.com",
    "timestamp": 1714329600,
    "smtp-id": "<x@y>",
    "event": "delivered",
    "sg_event_id": "sg_event_…",
    "sg_message_id": "Mid.…",
    "user_id": "42" },
  { "event": "open", "url": "https://...", "useragent": "...", ... },
  { "event": "click", "url": "https://...", ... }
]
```

Events: `processed`, `dropped`, `deferred`, `bounce`, `delivered`, `open`, `click`, `spamreport`, `unsubscribe`, `group_unsubscribe`, `group_resubscribe`.

⚠️ Verify the `X-Twilio-Email-Event-Webhook-Signature` (ECDSA) — different mechanism from Twilio webhooks. Use the SendGrid SDK's `EventWebhook.verify_signature(...)`.

## Inbound Parse

Receive emails into a webhook: configure MX records for a subdomain → SendGrid → POST multipart/form-data of each inbound message to your URL. Body fields: `from`, `to`, `subject`, `text`, `html`, `attachments` (count + numbered files), `headers`, `envelope`, `dkim`, `spf`.

Use case: support inboxes, ticket creation from email, AI assistants that read replies.

## Domain authentication

Improves deliverability. Add CNAME records SendGrid generates for **DKIM** (3 records), and either add an SPF include or let SendGrid manage it. Verified domains:

- Remove "via sendgrid.net" header.
- Sign emails with DKIM your domain owns.
- Required for DMARC alignment.

Without it: messages route as `*@em1234.example.com` (your "subdomain" CNAME); inboxing is fine but DMARC will fail strict policies.

## Marketing Campaigns (Marketing API v3)

Different surface from transactional Mail Send: lists / segments, automations, drag-drop designs. APIs hang off `/v3/marketing/...`. Worth a separate doc; for transactional alerts, stick with `/v3/mail/send`.

## Rate limits

- 100 emails/sec on the free tier; bumps with paid plans.
- Per-IP warm-up matters for new dedicated IPs — SendGrid offers "IP Warmup" automated ramps for big senders.
