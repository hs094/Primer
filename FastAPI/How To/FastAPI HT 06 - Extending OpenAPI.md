# HT 06 — Extending OpenAPI

🔑 Override `app.openapi` to add server objects, custom info, security schemes, vendor extensions — anything the auto-generator doesn't surface.

## Override pattern

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI()

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    schema = get_openapi(
        title="My API",
        version="2.0.0",
        summary="…",
        description="Long description with **markdown**",
        routes=app.routes,
        servers=[{"url": "https://api.example.com"}],
    )
    schema["info"]["x-logo"] = {"url": "https://example.com/logo.png"}
    app.openapi_schema = schema
    return schema

app.openapi = custom_openapi
```

⚠️ Cache the schema (`if app.openapi_schema: return ...`) — building it isn't free.

## Add a global security scheme

```python
schema["components"]["securitySchemes"] = {
    "bearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}
}
schema["security"] = [{"bearerAuth": []}]
```

## Per-path edits

After the base build, walk `schema["paths"]` and patch:

```python
schema["paths"]["/items/"]["get"]["x-rate-limit"] = "100/min"
```

Vendor extensions (`x-*`) are preserved by spec.

## Removing fields

Pydantic models can leak more than you want. Either prune in `custom_openapi`, or curate via `response_model_exclude=` / separate input/output models ([[FastAPI HT 07 - Separate OpenAPI Schemas Input Output]]).

## Reset on hot-reload

If you mutate after first `/openapi.json` request, set `app.openapi_schema = None` to force a rebuild.

💡 Keep the override small and declarative. If `custom_openapi` grows past ~30 lines, you're probably abusing it for runtime logic.
