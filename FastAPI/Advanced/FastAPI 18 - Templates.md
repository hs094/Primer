# Adv 18 — Templates (Jinja2)

🔑 FastAPI ⊂ Starlette → Starlette ships `Jinja2Templates`. Use it for SSR HTML when you don't want a separate frontend.

## Setup

```bash
pip install jinja2
```

```python
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app = FastAPI()
templates = Jinja2Templates(directory="templates")
app.mount("/static", StaticFiles(directory="static"), name="static")
```

## Render a template

```python
@app.get("/items/{id}", response_class=HTMLResponse)
async def get(request: Request, id: int):
    return templates.TemplateResponse(
        request=request,
        name="item.html",
        context={"id": id},
    )
```

⚠️ The `request` argument is mandatory — Jinja's `url_for` needs it.

## Template — `templates/item.html`

```jinja
<!DOCTYPE html>
<html>
<head><link rel="stylesheet" href="{{ url_for('static', path='/style.css') }}"></head>
<body><h1>Item {{ id }}</h1></body>
</html>
```

## When to use

- Internal tools where a SPA is overkill.
- Server-rendered emails (use `templates.get_template(...).render(...)` directly).
- Progressive enhancement (HTMX-style partials).

## Tips

- Set `autoescape=True` (default) to dodge XSS.
- Cache compiled templates in production by reusing the `Jinja2Templates` instance (FastAPI does by default).
- Custom filters: `templates.env.filters["money"] = lambda v: f"${v:,.2f}"`.

💡 If your "API" is really a website, this is the lightest path. Otherwise, lean SPA + JSON.
