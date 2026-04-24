# 23 — Metadata, Static Files, Templates

## App-level metadata

```python
app = FastAPI(
    title="My API",
    version="1.4.0",
    summary="Short desc",
    description="**Long** markdown description.",
    terms_of_service="https://example.com/tos",
    contact={"name": "Team", "url": "...", "email": "a@b.com"},
    license_info={"name": "MIT", "url": "https://opensource.org/licenses/MIT"},
    openapi_url="/api/openapi.json",   # None to disable
    docs_url="/docs",                  # None to disable Swagger
    redoc_url="/redoc",                # None to disable ReDoc
    root_path="/api/v1",               # for reverse-proxy prefix
)
```

## Tag metadata (order + descriptions)

```python
tags_metadata = [
    {"name": "users", "description": "User ops"},
    {"name": "items", "description": "Items",
     "externalDocs": {"description": "More", "url": "https://..."}},
]
app = FastAPI(openapi_tags=tags_metadata)
```

Controls order + descriptions of tag groups in docs.

## Static files

```python
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="static"), name="static")
```

- Serves `static/...` at `/static/...`.
- `StaticFiles(html=True)` serves `index.html` from directories.

## Templates (Jinja2)

```python
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse
from fastapi import Request

templates = Jinja2Templates(directory="templates")

@app.get("/items/{id}", response_class=HTMLResponse)
async def view(request: Request, id: int):
    return templates.TemplateResponse(
        request=request, name="item.html", context={"id": id}
    )
```

- `request` is required by Jinja context.
- Needs `pip install jinja2`.

## Gotchas

- ⚠️ Mount static routes **after** regular routes, or use a specific path prefix — mounted apps swallow everything under the mount.
- ⚠️ For SPA, use `html=True` + a catch-all route.
