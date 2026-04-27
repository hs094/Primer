# Deploy 04 — About FastAPI Versions

🔑 FastAPI follows **SemVer** but is pre-1.0 — at `0.MINOR.PATCH`, *minor* bumps can break, *patch* bumps are bug-fixes only. Pin a minor range in production.

## Recommended pinning

```txt
# requirements.txt / pyproject.toml dependencies
fastapi[standard]>=0.115.0,<0.116.0
```

Equivalent shorthands:

```toml
# pyproject.toml (PEP 621)
dependencies = ["fastapi[standard]~=0.115.0"]   # >=0.115.0,<0.116.0
```

For a stricter posture: `fastapi[standard]==0.115.4` (exact) and bump deliberately.

## What "minor" tends to mean

- New features **and** breaking changes (pre-1.0 convention).
- May pull Pydantic / Starlette upgrades transitively.
- Read the release notes — `https://fastapi.tiangolo.com/release-notes/`.

## What "patch" tends to mean

- Bug fixes, doc updates — no breaking changes.
- Safer to auto-update.

## Pydantic version

FastAPI ≥ 0.100 supports **Pydantic v2** natively (and is the recommended path). v1 still works via FastAPI's compat layer but is on its way out — migrate.

## Starlette version

FastAPI re-exports Starlette features and pins a compatible Starlette range itself — **don't pin Starlette directly**, let FastAPI manage it. A FastAPI bump may pull a Starlette upgrade; read both notes.

## Lockfile in CI/Docker

```bash
uv sync --frozen          # in CI
```

The lockfile (`uv.lock` / `poetry.lock`) is the contract — image rebuilds use exactly the same versions.

## Upgrade ritual

1. Read FastAPI + Pydantic + Starlette release notes for the range you're crossing.
2. Bump in `pyproject.toml`, regenerate lock.
3. Run full test suite, including [[FastAPI 20 - Testing Lifespan Events]].
4. Smoke-test in staging.
5. Tag release.

⚠️ Pre-1.0 `0.MINOR.PATCH` — treat *minor* like a major. Don't auto-merge dependabot PRs without reading notes.

💡 Pin tight in prod, loosen in dev to find regressions early.
