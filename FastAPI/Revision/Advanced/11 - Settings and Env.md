# Adv 11 — Settings & Env Vars (`pydantic-settings`)

🔑 One `Settings` class, instantiated once, injected via `Depends`. Types at boundaries, never raw `os.environ.get`.

## Install

```bash
pip install pydantic-settings
```

## Basic Settings

```python
# app/config.py
from functools import lru_cache
from pydantic import Field, PostgresDsn
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="",            # e.g. "APP_" → APP_DATABASE_URL
        extra="ignore",
    )

    app_name: str = "My API"
    debug: bool = False
    database_url: PostgresDsn
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str = Field(min_length=16)
    allowed_hosts: list[str] = []

@lru_cache
def get_settings() -> Settings:
    return Settings()             # parsed once, cached
```

`.env`:

```
DATABASE_URL=postgresql+asyncpg://user:pw@db/app
SECRET_KEY=change-me-please-0000
ALLOWED_HOSTS=["example.com","api.example.com"]
```

## Injecting in routes

```python
from fastapi import Depends
from typing import Annotated

SettingsDep = Annotated[Settings, Depends(get_settings)]

@app.get("/info")
async def info(settings: SettingsDep):
    return {"app": settings.app_name}
```

## Test-time override

```python
from app.config import get_settings

def test_settings():
    return Settings(database_url="sqlite+aiosqlite:///:memory:", secret_key="x"*16)

app.dependency_overrides[get_settings] = test_settings
```

## Nested / grouped settings

```python
class LoggingSettings(BaseSettings):
    level: str = "INFO"

class Settings(BaseSettings):
    logging: LoggingSettings = LoggingSettings()
```

With env_nested_delimiter: `LOGGING__LEVEL=DEBUG`.

```python
model_config = SettingsConfigDict(env_nested_delimiter="__")
```

## Secrets directory (Docker / Kubernetes)

```python
model_config = SettingsConfigDict(secrets_dir="/run/secrets")
```

Then `/run/secrets/database_url` is read as the field.

## Gotchas

- ⚠️ Multiple `Settings()` instantiations re-parse env and can silently disagree. Always go through `get_settings()`.
- ⚠️ `env_file` works locally but in prod **real env vars win**. Don't rely on `.env` in container images; ship env vars.
- ⚠️ Complex types (lists, dicts) in `.env` need JSON syntax.
