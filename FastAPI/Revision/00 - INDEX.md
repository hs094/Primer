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
| 01 | Install, first app, dev server, auto docs | [[Tutorial/01 - First Steps]] |
| 02 | Path parameters, types, Enum, `:path` converter | [[Tutorial/02 - Path Parameters]] |
| 03 | Query parameters, optionals, lists, bools | [[Tutorial/03 - Query Parameters]] |
| 04 | Request body with Pydantic models | [[Tutorial/04 - Request Body]] |
| 05 | `Query()` / `Path()` validation & metadata | [[Tutorial/05 - Query and Path Validation]] |
| 06 | Pydantic models as query/header/cookie groups | [[Tutorial/06 - Parameter Models]] |
| 07 | Multiple body params, `Body()`, `embed` | [[Tutorial/07 - Body Multiple Params]] |
| 08 | `Field()`, nested models, deeply nested schemas | [[Tutorial/08 - Body Fields and Nested]] |
| 09 | Declaring examples in schema + JSON Schema | [[Tutorial/09 - Declare Examples]] |
| 10 | Extra data types (datetime, UUID, bytes, …) | [[Tutorial/10 - Extra Data Types]] |
| 11 | Cookies & headers (single + model form) | [[Tutorial/11 - Cookies and Headers]] |
| 12 | `response_model`, filtering, inheritance | [[Tutorial/12 - Response Model]] |
| 13 | Extra models, `Union`, list/dict responses, status codes | [[Tutorial/13 - Extra Models and Status Codes]] |
| 14 | Forms, files, `UploadFile` vs `bytes` | [[Tutorial/14 - Forms and Files]] |
| 15 | Error handling, `HTTPException`, custom handlers | [[Tutorial/15 - Error Handling]] |
| 16 | Path op config, `jsonable_encoder`, PATCH updates | [[Tutorial/16 - Path Op Config and Encoder]] |
| 17 | Dependency injection with `Depends()` | [[Tutorial/17 - Dependencies]] |
| 18 | Security: OAuth2, JWT, scopes, API keys | [[Tutorial/18 - Security]] |
| 19 | Middleware & CORS | [[Tutorial/19 - Middleware and CORS]] |
| 20 | SQL databases with SQLModel / async SQLAlchemy | [[Tutorial/20 - SQL Databases]] |
| 21 | Bigger apps — `APIRouter` split | [[Tutorial/21 - Bigger Applications]] |
| 22 | Background tasks | [[Tutorial/22 - Background Tasks]] |
| 23 | Metadata, static files, templates | [[Tutorial/23 - Metadata Static Templates]] |
| 24 | Testing, debugging | [[Tutorial/24 - Testing and Debugging]] |

## Advanced User Guide

| # | Topic | Note |
|---|---|---|
| 01 | Path op advanced config, `openapi_extra` | [[Advanced/01 - Path Op Advanced Config]] |
| 02 | Custom responses (`HTMLResponse`, streaming, …) | [[Advanced/02 - Custom Responses]] |
| 03 | Additional responses, headers, cookies | [[Advanced/03 - Additional Responses]] |
| 04 | Advanced dependencies (parametric, classes) | [[Advanced/04 - Advanced Dependencies]] |
| 05 | Advanced security (scopes, OAuth2 flows) | [[Advanced/05 - Advanced Security]] |
| 06 | `Request`, dataclasses, `Annotated` patterns | [[Advanced/06 - Request and Dataclasses]] |
| 07 | Lifespan events — startup/shutdown | [[Advanced/07 - Lifespan Events]] |
| 08 | Sub-apps, `mount`, behind a proxy | [[Advanced/08 - Sub Apps and Proxy]] |
| 09 | WebSockets, SSE | [[Advanced/09 - WebSockets]] |
| 10 | Generate clients, OpenAPI extras | [[Advanced/10 - Generate Clients]] |
| 11 | Settings & env vars (`pydantic-settings`) | [[Advanced/11 - Settings and Env]] |
| 12 | Async tests with `httpx.AsyncClient` | [[Advanced/12 - Async Tests]] |

## Deployment

| # | Topic | Note |
|---|---|---|
| 01 | Concepts: HTTPS, workers, memory, restarts | [[Deployment/01 - Concepts]] |
| 02 | Running server, Uvicorn workers | [[Deployment/02 - Server Workers]] |
| 03 | Docker, multi-stage, workers-per-container | [[Deployment/03 - Docker]] |

## CLI

- [[CLI]] — `fastapi dev`, `fastapi run`

## Cross-links (already in this vault)

- [[FastAPI - Concurrency and Async Await]]
- [[Why should you use Depends() instead of declaring just a Global Instance?]]
