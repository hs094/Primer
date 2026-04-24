# FastAPI Revision Pack

Crisp, scannable revision notes covering the full FastAPI docs (tiangolo.com).
Source: https://fastapi.tiangolo.com/

## How to use
- Each note = single concept, optimized for recall, not reading-order.
- Syntax blocks are the minimum code that exercises the feature.
- 🔑 = core idea. ⚠️ = gotcha. 💡 = mental model. 🧪 = testable habit.

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
| 12 | `response_model`, filtering, inheritance | [[Tutorial/FastAPI 12 - Response Model]] |
| 13 | Extra models, `Union`, list/dict responses, status codes | [[Tutorial/FastAPI 13 - Extra Models and Status Codes]] |
| 14 | Forms, files, `UploadFile` vs `bytes` | [[Tutorial/FastAPI 14 - Forms and Files]] |
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

## Advanced User Guide

| # | Topic | Note |
|---|---|---|
| 01 | Path op advanced config, `openapi_extra` | [[Advanced/FastAPI 01 - Path Op Advanced Config]] |
| 02 | Custom responses (`HTMLResponse`, streaming, …) | [[Advanced/FastAPI 02 - Custom Responses]] |
| 03 | Additional responses, headers, cookies | [[Advanced/FastAPI 03 - Additional Responses]] |
| 04 | Advanced dependencies (parametric, classes) | [[Advanced/FastAPI 04 - Advanced Dependencies]] |
| 05 | Advanced security (scopes, OAuth2 flows) | [[Advanced/FastAPI 05 - Advanced Security]] |
| 06 | `Request`, dataclasses, `Annotated` patterns | [[Advanced/FastAPI 06 - Request and Dataclasses]] |
| 07 | Lifespan events — startup/shutdown | [[Advanced/FastAPI 07 - Lifespan Events]] |
| 08 | Sub-apps, `mount`, behind a proxy | [[Advanced/FastAPI 08 - Sub Apps and Proxy]] |
| 09 | WebSockets, SSE | [[Advanced/FastAPI 09 - WebSockets]] |
| 10 | Generate clients, OpenAPI extras | [[Advanced/FastAPI 10 - Generate Clients]] |
| 11 | Settings & env vars (`pydantic-settings`) | [[Advanced/FastAPI 11 - Settings and Env]] |
| 12 | Async tests with `httpx.AsyncClient` | [[Advanced/FastAPI 12 - Async Tests]] |

## Deployment

| # | Topic | Note |
|---|---|---|
| 01 | Concepts: HTTPS, workers, memory, restarts | [[Deployment/FastAPI 01 - Concepts]] |
| 02 | Running server, Uvicorn workers | [[Deployment/FastAPI 02 - Server Workers]] |
| 03 | Docker, multi-stage, workers-per-container | [[Deployment/FastAPI 03 - Docker]] |

## CLI

- [[FastAPI - CLI]] — `fastapi dev`, `fastapi run`

## Cross-links (already in this vault)

- [[FastAPI - Concurrency and Async Await]]
- [[FastAPI - Why Use Depends vs Global Instance]]
- [[FastAPI - Resources Index]] — external guides & articles
