# EX 02 — Web and API Requests

🔑 **Key insight:** treat every HTTP response body as untrusted JSON — `model_validate(response.json())` (or a `TypeAdapter` for collections) gives you a typed object and a hard schema boundary in one line.

## Why validate responses

Servers lie, change schemas, return partial data, or 200 with an error envelope. A Pydantic model in front of `httpx` (or `requests`, or anything that yields a dict) catches the drift at the edge instead of letting `KeyError`s and `None.foo` sprinkle through your code. See [[Pydantic CO 01 - Models]] for the model basics.

## Single record with httpx

`httpx` is the modern HTTP client with sync and async APIs. Validate a single user from JSONPlaceholder:

```python
import httpx

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


url = 'https://jsonplaceholder.typicode.com/users/1'

response = httpx.get(url)
response.raise_for_status()

user = User.model_validate(response.json())
print(repr(user))
#> User(id=1, name='Leanne Graham', email='[email protected]')
```

`raise_for_status()` handles transport-level failures; `model_validate` handles payload-shape failures. Two distinct error classes, two distinct boundaries.

## Lists with TypeAdapter

You cannot call `User.model_validate` on a list — the model is a single record. For collections, build a [[Pydantic CO 14 - Type Adapter]] over `list[User]`:

```python
from pprint import pprint

import httpx

from pydantic import BaseModel, EmailStr, TypeAdapter


class User(BaseModel):
  id: int
  name: str
  email: EmailStr


url = 'https://jsonplaceholder.typicode.com/users/'  # (1)

response = httpx.get(url)
response.raise_for_status()

users_list_adapter = TypeAdapter(list[User])

users = users_list_adapter.validate_python(response.json())
pprint([u.name for u in users])
"""
['Leanne Graham',
'Ervin Howell',
'Clementine Bauch',
'Patricia Lebsack',
'Chelsey Dietrich',
'Mrs. Dennis Schulist',
'Kurtis Weissnat',
'Nicholas Runolfsdottir V',
'Glenna Reichert',
'Clementina DuBuque']
"""
```

(1) The `/users/` endpoint returns a list — that is why we need the adapter rather than `User.model_validate`.

## When to use `validate_json` vs `validate_python`

`response.json()` already parses to Python. If you have raw bytes (`response.content`) or a string, prefer `validate_json` — the parser fuses with validation and skips an intermediate dict allocation:

```python
users = users_list_adapter.validate_json(response.content)
```

Use `validate_python` when something else has already deserialized the payload.

## Sending models out

Going the other way — building a request body from a model — use `model_dump` (or `model_dump_json`) from [[Pydantic CO 09 - Serialization]]:

```python
new_user = User(id=11, name='Ada Lovelace', email='ada@example.com')
httpx.post(url, json=new_user.model_dump())
```

`model_dump_json` returns a string ready for `content=`, skipping httpx's internal re-serialization.

## Framework integrations

Pydantic plugs into FastAPI (request bodies and response models are Pydantic), Django (via Pydantic-aware serializers), Flask, and HTTPX. The pattern is always the same: model in for inbound, model out for outbound.

⚠️ Validation runs eagerly — large list responses validate every item before you can iterate. For huge payloads, consider streaming the JSON and validating per-record, or use `TypeAdapter` with explicit error handling so one bad row does not nuke the whole batch (see [[Pydantic ER 02 - Validation Errors]]).

## Pattern recap

| Need | Call |
|---|---|
| Single record from `dict` | `Model.model_validate(payload)` |
| Single record from JSON string/bytes | `Model.model_validate_json(raw)` |
| List of records from `dict` | `TypeAdapter(list[Model]).validate_python(payload)` |
| List of records from JSON | `TypeAdapter(list[Model]).validate_json(raw)` |
| Build request body | `model.model_dump()` or `model.model_dump_json()` |

💡 **Takeaway:** every external HTTP boundary deserves a Pydantic model — validate inbound, dump outbound, never trust raw `response.json()`.
