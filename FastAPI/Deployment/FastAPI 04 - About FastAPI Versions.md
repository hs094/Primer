# Deploy 04 — About FastAPI Versions

🔑 FastAPI follows **SemVer-ish** but pre-1.0 — patch bumps can rarely include behavior tweaks. Pin a minor version range in production.

## Recommended pinning

```toml
# pyproject.toml
fastapi = "~0.115"          # >=0.115,<0.116  — patches only
# or
fastapi = "^0.115"          # >=0.115,<0.116  in poetry/uv (pre-1.0 behavior)
```

For a stricter posture: `fastapi==0.115.4` (exact) and bump deliberately.

## What "minor" tends to mean

- New features, occasional behavioral fixes.
- Pydantic / Starlette upgrades.
- Read the release notes — `https://fastapi.tiangolo.com/release-notes/`.

## What "patch" tends to mean

- Bug fixes, doc updates.
- Safer to auto-update.

## Pydantic version

FastAPI ≥ 0.100 supports **Pydantic v2** natively (and is the recommended path). v1 still works via FastAPI's compat layer but is on its way out — migrate.

## Starlette version

FastAPI re-exports Starlette features. A FastAPI bump may pull a Starlette upgrade — read both notes.

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
