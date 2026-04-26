# Environment Variables

🔑 Env vars are the OS-level config knobs that surround your process. Read once, type-validate via `pydantic-settings`, never sprinkle `os.environ.get(...)` through code.

## Reading raw

```python
import os
name = os.getenv("NAME", "World")  # default fallback
```

## `PATH` — special

`PATH` lists directories the shell searches for executables. The active virtualenv prepends its `bin/` so `python`, `uvicorn`, `fastapi` resolve to your project's binaries.

## Setting (shell)

```bash
export DATABASE_URL=postgres://...     # current shell + children
unset DATABASE_URL
DATABASE_URL=... uvicorn main:app      # one command only
```

## `.env` files

Plain `KEY=VALUE` per line, no `export`. Load via `pydantic-settings`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str
    debug: bool = False
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
```

⚠️ `.env` is *not* loaded by Python automatically. Either use `pydantic-settings` (preferred) or `python-dotenv` at process start.

⚠️ Never commit `.env` to git. Commit a `.env.example` instead.

## Typing the values

Env vars are *always strings* at the OS level. `pydantic-settings` parses to `int`, `bool`, `list[str]`, etc., based on your annotations.

```python
class Settings(BaseSettings):
    port: int            # "8000" → 8000
    debug: bool          # "true" / "1" / "yes" → True
    allowed_hosts: list[str]  # JSON-style or CSV
```

💡 Pair with [[FastAPI 11 - Settings and Env]] for the FastAPI-side wiring.
