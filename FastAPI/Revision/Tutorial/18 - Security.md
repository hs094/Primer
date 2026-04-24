# 18 — Security (OAuth2, JWT, API keys)

🔑 FastAPI ships security schemes that both *parse* the credential **and** register in OpenAPI so `/docs` shows the "Authorize" button.

## OAuth2 Password + Bearer (typical JWT setup)

```python
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from typing import Annotated

oauth2 = OAuth2PasswordBearer(tokenUrl="token")

@app.post("/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]):
    user = authenticate_user(form.username, form.password)
    if not user:
        raise HTTPException(401, "Bad creds",
                            headers={"WWW-Authenticate": "Bearer"})
    token = create_access_token(sub=user.username)
    return {"access_token": token, "token_type": "bearer"}

async def get_current_user(token: Annotated[str, Depends(oauth2)]):
    user = decode_token(token)        # raises 401 if invalid
    return user

@app.get("/me")
async def me(user: Annotated[User, Depends(get_current_user)]):
    return user
```

- `tokenUrl` is **relative** and is only metadata for docs.
- `OAuth2PasswordRequestForm` expects `username`/`password` as form fields.
- Always raise 401 with `WWW-Authenticate: Bearer` on failure.

## JWT — recommended libs

```bash
pip install "python-jose[cryptography]" "passlib[bcrypt]"
# or: pip install pyjwt bcrypt
```

```python
from jose import jwt, JWTError
SECRET = "..."
ALGO = "HS256"
EXP_MIN = 30

def create_access_token(sub: str):
    to_encode = {"sub": sub, "exp": datetime.utcnow() + timedelta(minutes=EXP_MIN)}
    return jwt.encode(to_encode, SECRET, algorithm=ALGO)

def decode_token(token: str) -> User:
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGO])
        username = payload.get("sub")
    except JWTError:
        raise HTTPException(401, "Invalid token",
                            headers={"WWW-Authenticate": "Bearer"})
    return get_user_by_username(username)
```

## Scopes

```python
oauth2 = OAuth2PasswordBearer(tokenUrl="token",
                              scopes={"me": "Read own info", "items": "Read items"})

async def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2)],
): ...

@app.get("/me")
async def me(user: Annotated[User, Security(get_current_user, scopes=["me"])]):
    ...
```

- Use `Security(...)` instead of `Depends(...)` when passing `scopes=`. Details in [[Advanced/05 - Advanced Security]].

## API Key

```python
from fastapi.security import APIKeyHeader
api_key_header = APIKeyHeader(name="X-API-Key")

async def verify(key: Annotated[str, Depends(api_key_header)]):
    if key != "expected": raise HTTPException(403, "Bad key")
```

Variants: `APIKeyQuery`, `APIKeyCookie`.

## HTTP Basic

```python
from fastapi.security import HTTPBasic, HTTPBasicCredentials
import secrets

basic = HTTPBasic()

def user(creds: Annotated[HTTPBasicCredentials, Depends(basic)]):
    ok_u = secrets.compare_digest(creds.username, "alice")
    ok_p = secrets.compare_digest(creds.password, "pw")
    if not (ok_u and ok_p):
        raise HTTPException(401, headers={"WWW-Authenticate": "Basic"})
```

⚠️ Always use `secrets.compare_digest` to prevent timing attacks.

## Gotchas

- ⚠️ Never log tokens / raw credentials.
- ⚠️ Rotate secrets via env; never commit them.
- 💡 Put auth as a global or router-level dependency for all protected routes; don't duplicate `Depends(get_current_user)` on every handler.
