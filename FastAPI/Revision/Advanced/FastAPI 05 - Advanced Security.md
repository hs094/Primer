# Adv 05 — Advanced Security

## `Security(...)` vs `Depends(...)`

Use `Security` when declaring **scopes**:

```python
from fastapi import Security
from fastapi.security import SecurityScopes, OAuth2PasswordBearer
from jose import jwt

oauth2 = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={"me": "Read own info", "items": "Read items"},
)

async def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2)],
) -> User:
    authenticate_value = (
        f'Bearer scope="{security_scopes.scope_str}"'
        if security_scopes.scopes else "Bearer"
    )
    credentials_exception = HTTPException(401, "Could not validate",
                                          headers={"WWW-Authenticate": authenticate_value})
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGO])
        username = payload.get("sub")
        token_scopes = payload.get("scopes", [])
    except JWTError:
        raise credentials_exception
    user = get_user(username)
    if user is None:
        raise credentials_exception
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(403, "Not enough permissions",
                                headers={"WWW-Authenticate": authenticate_value})
    return user

@app.get("/me")
async def me(user: Annotated[User, Security(get_current_user, scopes=["me"])]):
    return user

@app.get("/items/")
async def items(user: Annotated[User, Security(get_current_user, scopes=["items"])]):
    ...
```

## Alternatives to OAuth2 password flow

- **OAuth2 Authorization Code** — end-user browser flow via identity provider (Auth0, Google).
- **OpenID Connect** — adds identity on top of OAuth2.
- FastAPI provides `OAuth2AuthorizationCodeBearer`, `SecurityBase` for custom schemes.

## HTTP schemes

- `HTTPBearer` — plain `Authorization: Bearer <token>` (no OAuth2 flow metadata).
- `HTTPBasic`, `HTTPDigest` — standard HTTP auth.

## Combining auth schemes

Multiple `Security()` deps in the same route require **all** to pass.

```python
async def route(
    user: Annotated[User, Security(get_current_user, scopes=["me"])],
    _admin: Annotated[None, Security(require_admin)],
): ...
```

## Rate limiting & CSRF

Not built into FastAPI — use middleware or libs:
- `slowapi` / `fastapi-limiter` for rate limiting.
- `starlette-csrf` or cookie+header pattern for CSRF.

## Gotchas

- ⚠️ For public APIs protect routes with **global** / router-level security dependencies; easy to forget per-route.
- ⚠️ Always use `secrets.compare_digest` for credential equality checks to avoid timing leaks.
- ⚠️ Never expose validation internals via `detail` on auth errors — use generic "Invalid credentials".
