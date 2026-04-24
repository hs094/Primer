# Adv 08 — Sub-Apps, Mounts, Behind a Proxy

## Mounting a sub-FastAPI app

```python
from fastapi import FastAPI

app = FastAPI()
subapi = FastAPI()

@subapi.get("/sub")
def sub_root(): return {"app": "sub"}

app.mount("/subapi", subapi)
```

- `subapi` owns its own `/docs`, `/openapi.json` under `/subapi/...`.
- Root `app.openapi` **does not include** mounted sub-apps.
- Use for isolating internal tooling, different auth, different docs.

## Behind a reverse proxy — `root_path`

When a proxy serves your app at `/api/v1`, Uvicorn only sees `/`. To make docs + OpenAPI URLs correct:

```bash
uvicorn main:app --root-path /api/v1
# or
fastapi run main.py --root-path /api/v1
```

Or in code:

```python
app = FastAPI(root_path="/api/v1")
```

Effect:
- `/docs` served at `/api/v1/docs`.
- OpenAPI `servers:` populated with the root path.
- Path operations keep their internal (`/users`) paths.

## Custom `servers` in OpenAPI

```python
app = FastAPI(servers=[
    {"url": "https://stag.example.com", "description": "Staging"},
    {"url": "https://example.com", "description": "Production"},
])
```

Use with `root_path_in_servers=False` if you don't want the root_path prefix added on top.

## Proxy headers (HTTPS, client IP)

Uvicorn honors `X-Forwarded-*` when launched with `--proxy-headers` and `--forwarded-allow-ips=*`. Make sure your proxy sets them.

```bash
uvicorn main:app --proxy-headers --forwarded-allow-ips="*"
```

Then `request.url.scheme == "https"` behind a TLS terminator.

## Gotchas

- ⚠️ Without `root_path`, Swagger's "Try it out" will hit wrong URLs when behind a prefix proxy.
- ⚠️ Mounted Starlette/WSGI apps don't appear in the root OpenAPI.
- ⚠️ `--forwarded-allow-ips="*"` is fine behind a trusted proxy only — don't expose the ASGI server directly.
