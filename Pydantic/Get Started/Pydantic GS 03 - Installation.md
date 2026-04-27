# GS 03 — Installation

🔑 **Key insight:** `uv add pydantic` (or `pip install pydantic`) is all you need — Rust core, type backports, and constraint types come bundled.

## Requirements

- Python **3.9+**.
- `pip` or, preferably, [`uv`](https://github.com/astral-sh/uv).

> If you've got Python 3.9+ and `pip` installed, you're good to go.

## Install

```bash
pip install pydantic
```

```bash
uv add pydantic
```

That's it. No optional pieces are required for everyday `BaseModel` usage.

## What gets installed

Pydantic depends on three packages, all installed automatically:

- **`pydantic-core`** — core validation logic, written in Rust. Where the speed comes from.
- **`typing-extensions`** — backports newer `typing` features so V2 works on 3.9+.
- **`annotated-types`** — small library of constraint markers used with `typing.Annotated` (e.g. `Gt`, `Le`, `MinLen`).

Architecture deep-dive in [[Pydantic IT 01 - Architecture]].

## Optional extras

Two extras ship with Pydantic:

| Extra | Pulls in | Used by |
|---|---|---|
| `email` | `email-validator` | `EmailStr`, `NameEmail` ([[Pydantic CO 05 - Types]]) |
| `timezone` | `tzdata` | IANA timezone validation, mainly on Windows |

Install with:

```bash
pip install 'pydantic[email,timezone]'
```

```bash
uv add 'pydantic[email,timezone]'
```

Or install the underlying packages yourself:

```bash
pip install email-validator tzdata
```

## Conda

```bash
conda install pydantic -c conda-forge
```

## Install from source

For bleeding-edge or to test a fix on `main`:

```bash
pip install 'git+https://github.com/pydantic/pydantic@main'
```

⚠️ `main` is *not* covered by the V2 stability guarantees in [[Pydantic GS 05 - Version Policy]]. Pin to a release tag for anything you ship.

## Verify

```python
import pydantic
print(pydantic.VERSION)
```

If that prints a `2.x` string, you're set. V1 still ships as `pydantic.v1` inside V2 — see [[Pydantic GS 04 - Migration Guide]] for how that submodule works during gradual migrations.

## Sibling packages

Some V1 features moved out of the main package. Install separately when you need them:

```bash
uv add pydantic-settings        # BaseSettings — see [[Pydantic CO 17 - Settings Management]]
uv add pydantic-extra-types     # Color, PaymentCardNumber, etc.
```

💡 **Takeaway:** One command installs the Rust core; reach for extras only when you hit `EmailStr` or timezone work.
