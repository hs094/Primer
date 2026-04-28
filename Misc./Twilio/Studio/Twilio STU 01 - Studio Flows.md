# STU 01 — Studio Flows

🔑 Studio is a **drag-and-drop visual flow builder**. Use it for IVRs, basic chatbots, opt-in flows — anything where non-developers should iterate. Anything stateful, testable, or with complex business logic belongs in code (Functions or your own service) called via `Run Function` / `Make HTTP Request` widgets.

## Mental model

A **Flow** is a directed graph of **Widgets** with one **Trigger** entry. Each inbound message / call creates an **Execution** that walks the graph until it hits a stop widget or the trigger times out.

```
[Trigger]
   ├─ incomingMessage  ──→ widgets…
   ├─ incomingCall     ──→ widgets…
   └─ incomingRequest  ──→ widgets… (REST-triggered)
```

## Widget catalogue

### Messaging

| Widget | What it does |
|---|---|
| **Send Message** | One-shot SMS/WhatsApp send. No reply expected. |
| **Send & Wait For Reply** | Send, then pause execution until inbound reply. Outputs `incomingMessage` event. |

### Voice

| Widget | What it does |
|---|---|
| **Say/Play** | TwiML `<Say>` / `<Play>`. Voice only. |
| **Gather Input On Call** | TwiML `<Gather>` (DTMF and/or speech). Outputs `userSaidSomething`, `userPressedKeys`, `noInput`. |
| **Make Outgoing Call** | Place a new call. Useful as flow continuation. |
| **Connect Call To** | `<Dial>` — bridge to number, SIP, client, conference. |
| **Record Voicemail** | TwiML `<Record>`. |
| **Enqueue Call** | TwiML `<Enqueue>` into TaskRouter / Flex. |

### Logic / control

| Widget | What it does |
|---|---|
| **Split Based On…** | Branch via expression matching. Built-in regex and equality operators on any flow variable. |
| **Set Variables** | Assign one or more flow variables (Liquid-evaluated). |
| **Run Subflow** | Call another Flow (modular reuse). Returns to caller. |
| **Run Function** | Synchronously invoke a Twilio Function (Node.js). Returns the function's return value. |
| **Make HTTP Request** | Call any external HTTP API. Outputs response body / status. |
| **Send To Flex** | Hand the conversation/call to a Flex agent. |

Each widget exposes **Transitions**: named outputs (e.g. `Reply Received`, `No Reply` for *Send & Wait*) you draw arrows from. The flow editor enforces graph wiring.

## Liquid templating

Anywhere a widget takes a value, you can write Liquid: `{{trigger.message.From}}`, `{{widgets.greet.inbound.Body}}`, `{{flow.variables.counter | plus: 1}}`. Available scopes:

| Scope | Holds |
|---|---|
| `trigger` | Inbound event data (the SMS / call / request). |
| `widgets.<name>` | The output of the named widget after it ran. |
| `flow.variables` | Variables you've `Set Variables`'d. |
| `flow.data` | Deprecated synonym of variables. |
| `contact.channel.address` | Per-customer state (set via API). |

⚠️ Liquid evaluation is sandboxed — no arbitrary code. For anything beyond simple string/number ops, use a **Run Function** widget.

## Triggering flows

### Inbound SMS / call (most common)

Attach the flow's `webhook URL` to a phone number's `voice_url` / `sms_url`, or to a Messaging Service. Console → Phone Numbers → choose number → Voice/Messaging → "A call/message comes in" → "Studio Flow" → pick flow.

### REST-trigger (`incomingRequest`)

```python
exec = client.studio.v2.flows("FW...").executions.create(
    to="+15551234567",
    from_="+15557654321",
    parameters={"order_id": "1234", "amount": "42.00"},
)
print(exec.sid)        # FN...
```

Useful for outbound campaigns: kick a flow per customer, with custom parameters available as `{{flow.data.order_id}}`.

### Subflow

A Flow whose Trigger is `Run Subflow Trigger`. Receives params from caller; returns a result.

## Executions REST API

```python
# Inspect state
e = client.studio.v2.flows(flow_sid).executions(exec_sid).fetch()
print(e.status)        # active | ended

# Step history
for s in client.studio.v2.flows(flow_sid).executions(exec_sid).steps.list():
    print(s.name, s.transitioned_from, s.transitioned_to)

# Stop a flow
client.studio.v2.flows(flow_sid).executions(exec_sid).update(status="ended")

# Per-step context (Liquid scope at that step)
ctx = client.studio.v2.flows(flow_sid).executions(exec_sid).execution_context().fetch()
```

## Flow Definition (JSON)

A Flow is fundamentally a JSON document with `description`, `states`, and `initial_state`. Stored under version control:

```python
flow = client.studio.v2.flows.create(
    friendly_name="Order Updates",
    status="published",                 # or draft
    definition=json.dumps(flow_def),    # exported from UI or hand-edited
)
```

💡 Edit in UI for fast iteration, then export JSON and check into git for prod.

## Validation

```python
client.studio.v2.flow_validate.update(
    friendly_name="Order Updates", status="published", definition=json.dumps(flow_def),
)
```

Surfaces structural errors (orphan widgets, missing transitions) before publish.

## Limits worth remembering

- A *single* execution can run for at most **30 days** (then auto-ends).
- Each widget step has its own timeout — *Send & Wait For Reply* defaults to 1 hour, configurable up to a few hours.
- Flow ↔ Function calls are **synchronous and serial**; use *Make HTTP Request* with async upstream for long-running work.
- Studio is not free — pricing is per execution + per widget step. Heavy logic in Functions (free tier = 10k req/mo) is cheaper.

## When NOT to use Studio

- Programmatic workflows that need per-customer A/B logic, ML calls, or complex data joins → write code.
- Anything you need unit tests for — Flows have no test harness; only end-to-end execution traces.
- Multi-step state machines spanning days with cross-channel switching → build on Conversations + your own service.
