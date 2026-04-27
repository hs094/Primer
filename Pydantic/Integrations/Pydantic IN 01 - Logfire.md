# IN 01 — Logfire

🔑 **Key insight:** Logfire is built by the Pydantic team and ships first-class Pydantic instrumentation — one `logfire.instrument_pydantic()` call turns every model validation into a tracked span with success/failure detail and timing.

## Setup

Three steps to get Pydantic + Logfire wired up:

1. Set up a Logfire account
2. Install the Logfire SDK
3. Instrument your project

Install:

```bash
pip install logfire
```

Configure once at startup — usually next to your other observability bootstrapping (e.g. in your FastAPI lifespan):

```python
from datetime import date
import logfire
from pydantic import BaseModel

logfire.configure()  # Initialize Logfire

class User(BaseModel):
    name: str
    country_code: str
    dob: date

user = User(name='Anne', country_code='USA', dob='2000-01-01')
logfire.info('user processed: {user!r}', user=user)
```

`logfire.configure()` activates the SDK and sends data to your Logfire project; the SDK has built-in support for logging Pydantic models, so you can pass model instances directly into log messages.

## Auto-instrumenting every validation

For project-wide tracing without sprinkling `logfire.info` calls, opt in once:

```python
import logfire

logfire.instrument_pydantic()
```

The docs describe this as: it "automatically logs validation information for all Pydantic models in your project," capturing both successful validations and failures.

## What gets traced

Each call into `BaseModel(...)`, `Model.model_validate(...)`, `TypeAdapter.validate_python(...)`, etc. becomes a span carrying:

- successful model validations
- failed validations with full error details (the same payload as `ValidationError.errors()` — see [[Pydantic ER 01 - Error Handling]])
- timing and process information visible in Logfire spans

The Logfire UI lets you drill into individual validation events, see the failing input, and pivot on model name, error `type` ([[Pydantic ER 02 - Validation Errors]]), or duration.

## Where this fits

- Pair with [[Pydantic CO 17 - Settings Management|`pydantic-settings`]] to make sure your Logfire token / project comes from `BaseSettings` and not a stray env-var read.
- For [[Pydantic CO 14 - Type Adapter|`TypeAdapter`]]-heavy code paths (request bodies, queue payloads), `instrument_pydantic()` is the cheapest way to find which schemas are slow or noisy.
- Validation timing data is the same data point covered in [[Pydantic CO 18 - Performance]] — Logfire surfaces it in production rather than benchmarks.

⚠️ `instrument_pydantic()` traces *every* validation — for very hot paths (e.g. inner-loop `TypeAdapter` calls), measure overhead before turning it on globally; you may want to scope it to specific modules.

⚠️ Validation failures include the rejected `input`. If that input ever contains secrets or PII, scrub it at the Pydantic boundary (custom error handler) before it reaches Logfire.

## Manual logging of model instances

Even without `instrument_pydantic()`, you can log a model directly — Logfire knows how to render it:

```python
logfire.info('user processed: {user!r}', user=user)
```

This produces a structured attribute on the span keyed by `user`, with the model serialised by Pydantic itself, so field names match `model_dump()`.

💡 **Takeaway:** `logfire.configure()` + `logfire.instrument_pydantic()` is two lines for end-to-end visibility into every validation event in production.
