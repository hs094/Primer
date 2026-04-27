# CO 16 — Conversion Table

🔑 **Key insight:** Pydantic's coercion behaviour is a 5-axis matrix — **(field type, input type, mode, source, conditions)** — and lax-mode defaults are the source of most "why did that work?" surprises.

## The five axes

| Axis | Values |
|---|---|
| Field type | The annotated target (`bool`, `int`, `Decimal`, `datetime`, …) |
| Input type | The runtime type of the value being validated |
| Mode | `lax` (default) vs `strict` — see [[Pydantic CO 13 - Strict Mode]] |
| Input source | `Python` (`model_validate`) vs `JSON` (`model_validate_json`) |
| Conditions | Extra constraints (e.g. `val % 1 == 0`, `0 <= val <= 1`) |

The live filterable matrix lives at the upstream docs — this note covers the high-traffic conversions you'll actually hit.

## Booleans

Lax accepts a generous set; strict accepts only `bool`.

| Input | Mode | Source | Allowed values / conditions |
|---|---|---|---|
| `bool` | strict + lax | Python & JSON | always |
| `int` | lax | Python & JSON | only `0` and `1` |
| `float` | lax | Python & JSON | only `0.0` and `1.0` |
| `Decimal` | lax | Python | only `Decimal(0)` and `Decimal(1)` |
| `str` | lax | Python & JSON | `'0'`, `'off'`, `'f'`, `'false'`, `'n'`, `'no'`, `'1'`, `'on'`, `'t'`, `'true'`, `'y'`, `'yes'` (case-insensitive) |

⚠️ `bool` is `int`'s subclass in Python, but the int→bool path here is *value-restricted*, not just type-restricted: `Model(x=2)` for `x: bool` fails.

## Integers

| Input | Mode | Source | Conditions |
|---|---|---|---|
| `int` | strict + lax | Python & JSON | always |
| `float` | lax | Python & JSON | `val % 1 == 0`, rejects `nan` / `inf` |
| `Decimal` | lax | Python | `val % 1 == 0` |
| `bool` | lax | Python & JSON | `True`→1, `False`→0 |
| `str` | lax | Python & JSON | parseable as int |
| `bytes` | lax | Python | UTF-8 decodable, parseable as int |

## Floats

| Input | Mode | Source | Conditions |
|---|---|---|---|
| `float` | strict + lax | Python & JSON | always |
| `int` | strict + lax | Python & JSON | always |
| `Decimal` | lax | Python | always |
| `bool` | lax | Python & JSON | always |
| `str` | lax | Python & JSON | parseable as float |
| `bytes` | lax | Python | UTF-8 decodable, parseable as float |

## Strings & bytes

| Field | Input | Mode | Source | Notes |
|---|---|---|---|---|
| `str` | `str` | strict + lax | both | always |
| `str` | `bytes` | lax | Python | assumes UTF-8 |
| `str` | `bytearray` | lax | Python | assumes UTF-8 |
| `str` | `int`/`float`/`Decimal` | lax | Python | `str(val)` |
| `bytes` | `bytes` | strict + lax | both | always |
| `bytes` | `str` | lax | Python & JSON | UTF-8 encoded |
| `bytes` | `bytearray` | lax | Python | always |

## Decimal

| Input | Mode | Source |
|---|---|---|
| `Decimal` | strict + lax | Python |
| `int` | strict + lax | Python & JSON |
| `str` | strict + lax | Python & JSON (parseable) |
| `float` | lax | Python & JSON |
| `bool` | lax | Python & JSON |

## Date / datetime / time / timedelta

Strict accepts the native type plus ISO-8601 strings; lax additionally accepts numeric epoch values.

| Field | Input | Mode | Source | Conditions |
|---|---|---|---|---|
| `date` | `date` | strict + lax | Python | always |
| `date` | `str` | strict + lax | both | `YYYY-MM-DD` |
| `date` | `datetime` | lax | Python | midnight only |
| `date` | `int`/`float` | lax | Python & JSON | seconds since epoch, must align to day |
| `datetime` | `datetime` | strict + lax | Python | always |
| `datetime` | `str` | strict + lax | both | ISO-8601 / RFC 3339 |
| `datetime` | `int`/`float` | lax | Python & JSON | seconds since epoch |
| `time` | `time` | strict + lax | Python | always |
| `time` | `str` | strict + lax | both | `HH:MM[:SS[.ffffff]]` |
| `time` | `int`/`float` | lax | Python & JSON | seconds since midnight, `0 <= val < 86400` |
| `timedelta` | `timedelta` | strict + lax | Python | always |
| `timedelta` | `str` | strict + lax | both | ISO-8601 duration |
| `timedelta` | `int`/`float` | lax | Python & JSON | seconds |

## Sequences & mappings

| Field | Input (Python) | Input (JSON) | Strict |
|---|---|---|---|
| `list` | `list`, `tuple`, `set`, `frozenset`, `deque`, generators | array | only `list` |
| `tuple` | `list`, `tuple`, `set`, `frozenset`, `deque`, generators | array | only `tuple` |
| `set` / `frozenset` | `list`, `tuple`, `set`, `frozenset`, generators | array | only the matching type |
| `dict` | `dict`, mapping-likes | object | only `dict` |

⚠️ In **strict mode**, only the exact concrete type is accepted — strict `list` rejects a `tuple`, even though lax happily coerces. See [[Pydantic CO 13 - Strict Mode]].

## Network types

`IPv4Address`, `IPv6Address`, interfaces, networks:

| Input | Mode | Source |
|---|---|---|
| native ipaddress object | strict + lax | Python |
| `str` | strict + lax | Python & JSON |
| `int` | lax | Python only |
| `bytes` | lax | Python only |

## UUID, Path, Enum

| Field | Input | Source |
|---|---|---|
| `UUID` | `UUID`, `str` (any valid form), `bytes` (16-byte) | Python & JSON for `str` |
| `Path` | `Path`, `str` | both |
| `Enum` | enum member, member value, member name (lax) | both |

## JSON vs Python — the real differences

The matrix mostly differs along **Python-only inputs** (native `bytes`, `Decimal`, `datetime`, `set`, …) which simply don't exist in JSON. JSON validation:

- Parses strings into target types where applicable (ISO dates, UUIDs, IPs).
- Cannot receive a real `bytes` / `set` / `Decimal` — only their string/array representations.
- Treats objects as `dict` and arrays as `list` initially before coercion.

```python
from pydantic import BaseModel

class M(BaseModel):
    n: int
    d: list[int]

# Python: lax coerces
M.model_validate({'n': '42', 'd': (1, 2, 3)})

# JSON: same surface result, different code path
M.model_validate_json('{"n": "42", "d": [1, 2, 3]}')
```

## Strict mode — the short version

In strict mode the rule of thumb is:

- **Exact-type-only** for collections (`list` ≠ `tuple`).
- **No string→number** coercion in Python source (`'1'` for `int` fails).
- **String→date/UUID/IP** is still allowed in JSON source (since JSON has no native form).
- **`int`→`float` is preserved** even in strict.

Toggle per field with `Field(strict=True)`, per call with `model_validate(..., strict=True)`, or per model via `model_config = ConfigDict(strict=True)`. See [[Pydantic CO 08 - Configuration]].

## Per-field overrides

```python
from pydantic import BaseModel, Field

class M(BaseModel):
    coerce_me: int                          # lax: '42' → 42
    keep_strict: int = Field(strict=True)   # '42' → ValidationError
```

Combine with [[Pydantic CO 02 - Fields]] constraints (`gt`, `lt`, `pattern`, …) which apply *after* coercion.

## Where to look next

- [[Pydantic CO 05 - Types]] — what each annotation actually accepts.
- [[Pydantic CO 13 - Strict Mode]] — toggling lax vs strict.
- [[Pydantic CO 14 - Type Adapter]] — same coercion rules outside of `BaseModel`.
- [[Pydantic IT 01 - Architecture]] — why `pydantic-core` does this in Rust.

💡 **Takeaway:** Lax mode is "do what I mean" — strict mode is "do what I said." Pick deliberately; never assume.
