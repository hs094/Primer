# Editor Support

🔑 FastAPI's whole pitch is editor-driven. Type hints + Pydantic models = autocomplete, go-to-def, inline errors *for request data*, not just code.

## Officially tested

- **VS Code** (with Pylance / Pyright)
- **PyCharm** (Professional or Community)

Both surface:
- Path/query/body parameter completion.
- Pydantic field completion on `item.name`, `item.price`, etc.
- Type errors before you run.
- Inline docs from docstrings.

## Setup tips

- Point the editor's Python interpreter at your `.venv/bin/python`.
- Enable strict-ish type checking (`pyright: strict`, PyCharm "type checker" inspection).
- Install the **Ruff** extension for format + lint.

## Why it works

FastAPI deliberately avoids magic strings and dynamic configuration:

```python
@app.get("/items/{item_id}")
async def read(item_id: int, q: str | None = None):
    return ...
```

`item_id`, `q`, return type — all standard Python. The editor needs no FastAPI-specific plugin.

## Tooling pairings

- **Ruff** — format + lint (replaces black/isort/flake8).
- **Mypy** or **Pyright** — static type checking for stricter contracts.
- **`pytest`** with `httpx.AsyncClient` for handler-level tests.

💡 If your editor *can't* autocomplete a request parameter, you've probably typed it as `dict` instead of a `BaseModel` — that's the smell.
