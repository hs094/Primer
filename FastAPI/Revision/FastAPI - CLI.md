# FastAPI CLI

🔑 `fastapi` command ships with `pip install "fastapi[standard]"`. Wraps Uvicorn with sensible defaults.

## Commands

```bash
fastapi dev  [PATH]     # development: reload, host=127.0.0.1
fastapi run  [PATH]     # production-ish: no reload, host=0.0.0.0
fastapi --help
fastapi dev --help
```

- `PATH` defaults to `main.py` in CWD. Must expose a `FastAPI` instance named `app` (or pass `--app`).

## Common flags

| Flag | Meaning |
|---|---|
| `--app NAME` | variable name of your `FastAPI` instance |
| `--host 0.0.0.0` | bind address |
| `--port 8000` | port |
| `--reload` / `--no-reload` | auto-reload on file changes |
| `--workers N` | spawn N worker processes (`run` only) |
| `--root-path /api` | behind a prefix proxy |
| `--proxy-headers` | trust `X-Forwarded-*` |
| `--log-level info` | `debug` / `info` / `warning` / `error` |

## When to use vs direct Uvicorn

- `fastapi dev` / `fastapi run` — ergonomic, right defaults.
- `uvicorn ...` directly — maximum control, scripting, or when you don't install the `[standard]` extras.

## Examples

```bash
# dev with a non-default file and app name
fastapi dev src/api.py --app application

# prod: 4 workers behind a reverse proxy
fastapi run app/main.py --workers 4 --root-path /api --proxy-headers
```

## Gotcha

- ⚠️ Installing slim `fastapi` (no `[standard]`) won't include the CLI — import error at `fastapi dev`.
