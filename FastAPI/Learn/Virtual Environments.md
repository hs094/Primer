# Virtual Environments

🔑 One venv per project. Isolates dependencies so your FastAPI app's `pydantic==2.x` doesn't fight another tool's `pydantic==1.x`.

## With `uv` (preferred)

```bash
uv venv                       # creates .venv/
uv add fastapi[standard]      # installs + records in pyproject.toml
uv sync                       # match lockfile exactly
uv run uvicorn main:app       # runs in venv without activating
```

`uv` manages the venv, lockfile (`uv.lock`), and Python version itself.

## With stdlib `venv`

```bash
python -m venv .venv
source .venv/bin/activate     # Linux/macOS
.venv\Scripts\activate        # Windows
pip install "fastapi[standard]"
deactivate
```

## What activation does

Prepends `.venv/bin` to `PATH` and sets `VIRTUAL_ENV`. Without it, `python` resolves to the system interpreter and you'd install globally.

## Verify

```bash
which python    # → .../.venv/bin/python
python -c "import sys; print(sys.prefix)"
```

## Common pitfalls

- ⚠️ Don't commit `.venv/` (add to `.gitignore`).
- ⚠️ One venv per project, not one per developer-shared. Recreate from `pyproject.toml` / `uv.lock` / `requirements.txt`.
- ⚠️ Don't mix `pip` and `uv` in the same venv — pick one resolver.

## Lockfiles

- `uv.lock` (uv) → exact, cross-platform.
- `requirements.txt` via `pip freeze` → flat, no transitive resolution metadata.

💡 Lock for reproducibility, install from lock in CI/Docker.
