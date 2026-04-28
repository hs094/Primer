# SVL 01 — Functions and Assets

🔑 **Functions** = Twilio-hosted Node.js endpoints, ideal for the small bits of glue Studio can't do (or to keep secrets out of Studio). **Assets** = static files served from Twilio's CDN (audio for `<Play>`, JSON config, web pages).

## Mental model

```
Service (ZS…)                 ← deployment unit, contains many Functions/Assets
└── Environment (ZE…)         ← named env (dev / staging / prod), one URL per env
    ├── Build (ZB…)           ← snapshot of code + deps + env vars
    │   ├── Function Versions
    │   └── Asset Versions
    └── Deployment (ZD…)      ← currently-active Build for this Environment
Function (ZH…)                ← logical Function, has many versions
Asset (ZF…)                   ← logical Asset, has many versions
Variable (ZV…)                ← env var bound to an Environment
```

## Hello-world Function

```js
// functions/hello.js
exports.handler = function (context, event, callback) {
  const twiml = new Twilio.twiml.VoiceResponse();
  twiml.say(`Hi ${event.name || "stranger"}`);
  return callback(null, twiml);
};
```

Signature: `(context, event, callback)`.

| Arg | What it is |
|---|---|
| `context` | `{ ACCOUNT_SID, AUTH_TOKEN, getTwilioClient(), DOMAIN_NAME, PATH, SERVICE_SID, ENVIRONMENT_SID, <your env vars> }`. |
| `event` | Merged query string + form body (or JSON body). |
| `callback(err, value)` | Respond. `value` may be a string (`text/plain`), a JSON object (`application/json`), a `TwiML` builder, or a `Twilio.Response` for full control. |

Async style:

```js
exports.handler = async function (context, event) {
  const client = context.getTwilioClient();
  const msg = await client.messages.create({ to: event.to, from: event.from, body: "hi" });
  return { ok: true, sid: msg.sid };
};
```

(When the handler returns a `Promise`, you can omit the callback.)

## Twilio.Response — full HTTP control

```js
const response = new Twilio.Response();
response.setStatusCode(204);
response.appendHeader("Cache-Control", "no-store");
response.appendHeader("Access-Control-Allow-Origin", "*");
response.setBody({ ok: true });
return callback(null, response);
```

Use this for CORS, custom status codes, or non-JSON content types (XML for TwiML, etc.).

## Deps

`package.json` at the service root. The Serverless Toolkit / Console resolves dependencies at build time:

```json
{
  "dependencies": {
    "axios": "^1.6.0",
    "openai": "^4.0.0"
  }
}
```

⚠️ Bundle size cap is small (a few MB). Native modules / FFI are not supported.

## Visibility

Each Function (and each Asset) is `public` or `protected`:

| Visibility | Behaviour |
|---|---|
| `public` | Open to the internet at the Function's URL. |
| `protected` | Only callable when the request is signed by Twilio (i.e., from Studio, TwiML callbacks, or your own signed request). The runtime auto-validates. |
| `private` | Importable from other Functions in the same service (`require(Runtime.getFunctions()['utils'].path)`) but not exposed via HTTP. |

💡 Default everything to `protected` if it'll only ever be called from a TwiML / Studio webhook.

## Environment variables

Set per-environment (Console or `.env` via the toolkit):

```
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
OPENAI_API_KEY=sk-...
```

Inside the function: `context.OPENAI_API_KEY`. Never log them.

## Runtime helpers

```js
// Helper: get a configured Twilio client
const client = context.getTwilioClient();

// Helper: load a private function as a module
const utils = require(Runtime.getFunctions()['lib/utils'].path);

// Helper: load an asset path
const assetPath = Runtime.getAssets()['/voicemail.mp3'].path;
```

`Runtime.getSync()` exposes Twilio Sync — KV/state outside the function lifecycle (durable state without bringing your own DB).

## Assets

Drop files under `assets/` (or upload via Console). Public assets get a stable URL like `https://my-service-1234.twil.io/voicemail.mp3` — perfect for `<Play>` audio:

```xml
<Response>
  <Play>https://my-service-1234.twil.io/hold.mp3</Play>
</Response>
```

Protected assets serve only when called from Twilio (via signature). Useful for sensitive payloads referenced from TwiML.

## Logs & debugging

- Console → Functions → Logs (live tail).
- `console.log` / `console.error` end up in the log stream and Twilio Debugger.
- Errors auto-reported to **Twilio Debugger**. Set `Debugger Webhook` to forward incidents to your own monitoring.

## Local development — Serverless Toolkit

```bash
npm install -g @twilio-labs/plugin-serverless    # or: npx twilio plugins:install @twilio-labs/plugin-serverless
twilio serverless:init my-service
cd my-service
twilio serverless:start                          # local dev: http://localhost:3000
twilio serverless:deploy --environment=dev       # deploy
```

The local dev server emulates the runtime well enough for rapid iteration. For TwiML callbacks during local dev, expose with `ngrok` or `cloudflared`.

## Sample Functions

### Inbound SMS auto-responder

```js
exports.handler = (context, event, callback) => {
  const twiml = new Twilio.twiml.MessagingResponse();
  twiml.message(`Got your message: "${event.Body}"`);
  callback(null, twiml);
};
```

### REST endpoint that sends an OTP

```js
exports.handler = async (context, event) => {
  const client = context.getTwilioClient();
  const r = await client.verify.v2
    .services(context.VERIFY_SERVICE_SID)
    .verifications.create({ to: event.to, channel: "sms" });
  return { status: r.status };
};
```

## When to use Functions vs your own backend

- **Functions** good for: TwiML callbacks, light Studio glue, prototypes, no-infra-required teams.
- **Your own service** better for: anything > a few hundred lines, anything sharing logic with non-Twilio paths, anything you want unit-tested, anything where you want choice of runtime / language.

⚠️ Functions are billed per-invocation + per-GB-second. Cheap at low scale; significantly more expensive than self-hosted at high scale.

## Limits

- Max execution time: **10 seconds** (hard timeout, returns 504).
- Max payload: 5 MB request, 5 MB response.
- 30 concurrent invocations per service by default (raise via support).
- Cold start: ~100–500 ms first hit per environment.
