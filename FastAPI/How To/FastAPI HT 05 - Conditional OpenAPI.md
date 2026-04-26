# HT 05 — Conditional OpenAPI

🔑 Disable `/docs`, `/redoc`, and `/openapi.json` in production (or behind auth) by toggling URLs on the `FastAPI()` constructor based on settings.

## Disable in production

```python
from fastapi import FastAPI
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    env: str = "dev"

settings = Settings()

app = FastAPI(
    openapi_url="/openapi.json" if settings.env != "prod" else None,
    docs_url="/docs" if settings.env != "prod" else None,
    redoc_url="/redoc" if settings.env != "prod" else None,
)
```

Setting any of these to `None` removes the route.

## Auth-gated docs

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBasic, HTTPBasicCredentials
import secrets

security = HTTPBasic()

def check(creds: HTTPBasicCredentials = Depends(security)):
    user_ok = secrets.compare_digest(creds.username, "admin")
    pw_ok   = secrets.compare_digest(creds.password, settings.docs_password)
    if not (user_ok and pw_ok):
        raise HTTPException(401, headers={"WWW-Authenticate": "Basic"})
    return True

@app.get("/docs", include_in_schema=False)
async def custom_docs(_: bool = Depends(check)):
    return get_swagger_ui_html(openapi_url="/openapi.json", title="docs")
```

⚠️ If you customise `/docs`, set `docs_url=None` on the app constructor first or you'll route-collision.

## Why hide docs

- Surface area: docs reveal every internal endpoint, schema, parameter — attackers love it.
- Compliance / brand: customers shouldn't see admin/internal endpoints.
- Performance: docs aren't free to serve under load.

## Counterargument

- For internal-only services behind a VPN, leave docs on — devs love them.
- Public APIs *should* publish docs (separately, curated).

💡 Default to off in prod; opt in deliberately.
