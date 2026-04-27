# HT 08 — Custom Docs UI Static Assets (Self-Hosting)

🔑 Default `/docs` and `/redoc` pull JS/CSS from a CDN. Self-host the assets when you're behind a strict CSP, in an air-gapped network, or want predictable load times.

## Disable defaults, register custom routes

```python
from fastapi import FastAPI
from fastapi.openapi.docs import (
    get_redoc_html,
    get_swagger_ui_html,
    get_swagger_ui_oauth2_redirect_html,
)
from fastapi.staticfiles import StaticFiles

app = FastAPI(docs_url=None, redoc_url=None)
app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/docs", include_in_schema=False)
async def swagger():
    return get_swagger_ui_html(
        openapi_url=app.openapi_url,
        title=app.title + " - Swagger UI",
        oauth2_redirect_url=app.swagger_ui_oauth2_redirect_url,
        swagger_js_url="/static/swagger-ui-bundle.js",
        swagger_css_url="/static/swagger-ui.css",
    )

@app.get(app.swagger_ui_oauth2_redirect_url, include_in_schema=False)
async def swagger_ui_redirect():
    return get_swagger_ui_oauth2_redirect_html()

@app.get("/redoc", include_in_schema=False)
async def redoc():
    return get_redoc_html(
        openapi_url=app.openapi_url,
        title=app.title + " - ReDoc",
        redoc_js_url="/static/redoc.standalone.js",
    )
```

## Get the assets

- Swagger UI: `swagger-ui-dist` npm / GitHub releases — copy `swagger-ui-bundle.js` and `swagger-ui.css`.
- ReDoc: `redoc.standalone.js` from the GitHub release.

Vendor them into `static/` and pin versions.

## OAuth2 redirect (Swagger)

The redirect handler is registered at `app.swagger_ui_oauth2_redirect_url` (default `/docs/oauth2-redirect`). Pass the same URL into `get_swagger_ui_html(oauth2_redirect_url=...)` so Swagger knows where to send the OAuth2 flow callback. Both are wired in the snippet above.

## Why self-host

- Strict CSP: no third-party origins allowed.
- Air-gapped / offline.
- Reproducible builds — CDN went down won't break docs.
- Performance: served from same origin, cacheable headers under your control.

⚠️ Keep the assets up to date — security fixes do happen in Swagger UI / ReDoc.

💡 Vendor + pin. Renovate/Dependabot for the npm package can flag updates.
