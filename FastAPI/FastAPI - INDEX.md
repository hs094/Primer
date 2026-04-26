# FastAPI Knowledge Pack

Crisp, scannable revision notes covering the full FastAPI docs (tiangolo.com).
Source: https://fastapi.tiangolo.com/

## How to use
- Each note = one concept, optimized for recall.
- Code blocks are the minimum that exercises the feature.
- 🔑 = core idea. ⚠️ = gotcha. 💡 = mental model. 🧪 = testable habit.

## Learn (foundations)

| Topic | Note |
|---|---|
| Python type hints, `Annotated`, generics | [[Learn/Python Types Intro]] |
| Concurrency, `async`/`await`, threadpool | [[Learn/Concurrency and Async Await]] |
| Environment variables, `.env`, `pydantic-settings` | [[Learn/Environment Variables]] |
| Virtual environments, `uv`, `venv`, lockfiles | [[Learn/Virtual Environments]] |

## Tutorial — User Guide

| # | Topic | Note |
|---|---|---|
| 01 | Install, first app, dev server, auto docs | [[Tutorial/FastAPI 01 - First Steps]] |
| 02 | Path parameters, types, Enum, `:path` converter | [[Tutorial/FastAPI 02 - Path Parameters]] |
| 03 | Query parameters, optionals, lists, bools | [[Tutorial/FastAPI 03 - Query Parameters]] |
| 04 | Request body with Pydantic models | [[Tutorial/FastAPI 04 - Request Body]] |
| 05 | `Query()` / `Path()` validation & metadata | [[Tutorial/FastAPI 05 - Query and Path Validation]] |
| 06 | Pydantic models as query/header/cookie groups | [[Tutorial/FastAPI 06 - Parameter Models]] |
| 07 | Multiple body params, `Body()`, `embed` | [[Tutorial/FastAPI 07 - Body Multiple Params]] |
| 08 | `Field()`, nested models, deeply nested schemas | [[Tutorial/FastAPI 08 - Body Fields and Nested]] |
| 09 | Declaring examples in schema + JSON Schema | [[Tutorial/FastAPI 09 - Declare Examples]] |
| 10 | Extra data types (datetime, UUID, bytes, …) | [[Tutorial/FastAPI 10 - Extra Data Types]] |
| 11 | Cookies & headers (single + model form) | [[Tutorial/FastAPI 11 - Cookies and Headers]] |
| 11b | Cookie / header parameter models | [[Tutorial/FastAPI 11b - Cookie and Header Parameter Models]] |
| 12 | `response_model`, filtering, inheritance | [[Tutorial/FastAPI 12 - Response Model]] |
| 13 | Extra models, `Union`, list/dict responses, status codes | [[Tutorial/FastAPI 13 - Extra Models and Status Codes]] |
| 14 | Forms, files, `UploadFile` vs `bytes` | [[Tutorial/FastAPI 14 - Forms and Files]] |
| 14b | Form models | [[Tutorial/FastAPI 14b - Form Models]] |
| 15 | Error handling, `HTTPException`, custom handlers | [[Tutorial/FastAPI 15 - Error Handling]] |
| 16 | Path op config, `jsonable_encoder`, PATCH updates | [[Tutorial/FastAPI 16 - Path Op Config and Encoder]] |
| 17 | Dependency injection with `Depends()` | [[Tutorial/FastAPI 17 - Dependencies]] |
| 18 | Security: OAuth2, JWT, scopes, API keys | [[Tutorial/FastAPI 18 - Security]] |
| 19 | Middleware & CORS | [[Tutorial/FastAPI 19 - Middleware and CORS]] |
| 20 | SQL databases with SQLModel / async SQLAlchemy | [[Tutorial/FastAPI 20 - SQL Databases]] |
| 21 | Bigger apps — `APIRouter` split | [[Tutorial/FastAPI 21 - Bigger Applications]] |
| 22 | Background tasks | [[Tutorial/FastAPI 22 - Background Tasks]] |
| 23 | Metadata, static files, templates | [[Tutorial/FastAPI 23 - Metadata Static Templates]] |
| 24 | Testing, debugging | [[Tutorial/FastAPI 24 - Testing and Debugging]] |
| 25 | Stream JSON Lines (NDJSON) | [[Tutorial/FastAPI 25 - Stream JSON Lines]] |
| 26 | Server-Sent Events (SSE) | [[Tutorial/FastAPI 26 - Server-Sent Events]] |

## Advanced User Guide

| # | Topic | Note |
|---|---|---|
| 01 | Path op advanced config, `openapi_extra` | [[Advanced/FastAPI 01 - Path Op Advanced Config]] |
| 02 | Custom responses (`HTMLResponse`, streaming, …) | [[Advanced/FastAPI 02 - Custom Responses]] |
| 03 | Additional responses, headers, cookies | [[Advanced/FastAPI 03 - Additional Responses]] |
| 04 | Advanced dependencies (parametric, classes) | [[Advanced/FastAPI 04 - Advanced Dependencies]] |
| 05 | Advanced security (scopes, OAuth2 flows, HTTP Basic) | [[Advanced/FastAPI 05 - Advanced Security]] |
| 06 | `Request`, dataclasses, `Annotated` patterns | [[Advanced/FastAPI 06 - Request and Dataclasses]] |
| 07 | Lifespan events — startup/shutdown | [[Advanced/FastAPI 07 - Lifespan Events]] |
| 08 | Sub-apps, `mount`, behind a proxy | [[Advanced/FastAPI 08 - Sub Apps and Proxy]] |
| 09 | WebSockets, SSE | [[Advanced/FastAPI 09 - WebSockets]] |
| 10 | Generate clients / SDKs, OpenAPI extras | [[Advanced/FastAPI 10 - Generate Clients]] |
| 11 | Settings & env vars (`pydantic-settings`) | [[Advanced/FastAPI 11 - Settings and Env]] |
| 12 | Async tests with `httpx.AsyncClient` | [[Advanced/FastAPI 12 - Async Tests]] |
| 13 | Stream data — `StreamingResponse` | [[Advanced/FastAPI 13 - Stream Data]] |
| 14 | Additional status codes mid-handler | [[Advanced/FastAPI 14 - Additional Status Codes]] |
| 15 | Return a `Response` directly (bypass pipeline) | [[Advanced/FastAPI 15 - Return a Response Directly]] |
| 16 | Inject `Response`, mutate status / headers | [[Advanced/FastAPI 16 - Response Change Status Code]] |
| 17 | Advanced middleware (HTTPS, TrustedHost, GZip, custom) | [[Advanced/FastAPI 17 - Advanced Middleware]] |
| 18 | Server-rendered Jinja2 templates | [[Advanced/FastAPI 18 - Templates]] |
| 19 | Testing WebSockets with `TestClient` | [[Advanced/FastAPI 19 - Testing WebSockets]] |
| 20 | Testing lifespan / startup / shutdown | [[Advanced/FastAPI 20 - Testing Lifespan Events]] |
| 21 | Testing dependencies via `dependency_overrides` | [[Advanced/FastAPI 21 - Testing Dependency Overrides]] |
| 22 | OpenAPI callbacks (per-request webhook docs) | [[Advanced/FastAPI 22 - OpenAPI Callbacks]] |
| 23 | OpenAPI webhooks (global event subscriptions) | [[Advanced/FastAPI 23 - OpenAPI Webhooks]] |
| 24 | Including WSGI — Flask / Django mounts | [[Advanced/FastAPI 24 - Including WSGI - Flask Django]] |
| 25 | Advanced Python types, generics, `Literal`, `NewType` | [[Advanced/FastAPI 25 - Advanced Python Types]] |
| 26 | JSON with bytes as base64 | [[Advanced/FastAPI 26 - JSON with Bytes as Base64]] |
| 27 | Strict `Content-Type` checking | [[Advanced/FastAPI 27 - Strict Content-Type Checking]] |

## CLI

- [[CLI/FastAPI - CLI]] — `fastapi dev`, `fastapi run`

## Editor Support

- [[Editor Support/Editor Support]] — VS Code, PyCharm, Pylance/Pyright, Ruff

## Deployment

| # | Topic | Note |
|---|---|---|
| 01 | Concepts: HTTPS, workers, memory, restarts | [[Deployment/FastAPI 01 - Concepts]] |
| 02 | Running server, Uvicorn workers | [[Deployment/FastAPI 02 - Server Workers]] |
| 03 | Docker, multi-stage, workers-per-container | [[Deployment/FastAPI 03 - Docker]] |
| 04 | About FastAPI versions & pinning | [[Deployment/FastAPI 04 - About FastAPI Versions]] |
| 05 | FastAPI Cloud (managed PaaS) | [[Deployment/FastAPI 05 - FastAPI Cloud]] |
| 06 | Run a server manually (Uvicorn / Gunicorn) | [[Deployment/FastAPI 06 - Run a Server Manually]] |
| 07 | Deploy on cloud providers (AWS / GCP / k8s / VM) | [[Deployment/FastAPI 07 - Cloud Providers]] |

## How To — Recipes

| # | Topic | Note |
|---|---|---|
| 01 | Overview / when to reach for a recipe | [[How To/FastAPI HT 01 - General How To]] |
| 02 | Migrate Pydantic v1 → v2 | [[How To/FastAPI HT 02 - Migrate Pydantic v1 to v2]] |
| 03 | GraphQL via Strawberry / Ariadne | [[How To/FastAPI HT 03 - GraphQL]] |
| 04 | Custom `Request` / `APIRoute` class | [[How To/FastAPI HT 04 - Custom Request and APIRoute]] |
| 05 | Conditional OpenAPI (hide docs in prod) | [[How To/FastAPI HT 05 - Conditional OpenAPI]] |
| 06 | Extending the OpenAPI schema | [[How To/FastAPI HT 06 - Extending OpenAPI]] |
| 07 | Separate input vs output OpenAPI schemas | [[How To/FastAPI HT 07 - Separate OpenAPI Schemas Input Output]] |
| 08 | Custom docs UI static assets (self-host) | [[How To/FastAPI HT 08 - Custom Docs UI Static Assets]] |
| 09 | Configure Swagger UI parameters | [[How To/FastAPI HT 09 - Configure Swagger UI]] |
| 10 | Testing a real database | [[How To/FastAPI HT 10 - Testing a Database]] |
| 11 | Use old 403 auth status codes | [[How To/FastAPI HT 11 - Use Old 403 Auth Status Codes]] |

## Cross-links

- [[FastAPI - Resources Index]] — external guides & articles
- [[Thoughts/FastAPI - Why Use Depends vs Global Instance]]
