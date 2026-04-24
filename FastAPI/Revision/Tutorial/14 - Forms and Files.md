# 14 — Forms & Files

🔑 Needs `python-multipart` installed. `Form(...)` + `File(...)` are siblings of `Body(...)`.

## Form fields

```python
from fastapi import Form
from typing import Annotated

@app.post("/login/")
async def login(
    username: Annotated[str, Form()],
    password: Annotated[str, Form()],
):
    ...
```

Content-type: `application/x-www-form-urlencoded` or `multipart/form-data`.

## Single file (bytes vs UploadFile)

```python
from fastapi import File, UploadFile

@app.post("/upload-bytes/")
async def upload_bytes(file: Annotated[bytes, File()]):
    return {"size": len(file)}

@app.post("/upload/")
async def upload(file: UploadFile):
    return {"filename": file.filename, "type": file.content_type}
```

| `bytes` | `UploadFile` |
|---|---|
| Loads entire file into memory | Spooled: memory up to a threshold, then disk |
| Simpler for small payloads | Preferred for anything non-trivial |
| No metadata | `.filename`, `.content_type`, `.size`, `.file`, async `.read/.write/.seek/.close` |

## Optional file

```python
file: UploadFile | None = None
```

## Multiple files

```python
files: list[UploadFile]
files: Annotated[list[bytes], File()]
```

## Form + files together

```python
async def f(
    token: Annotated[str, Form()],
    file: UploadFile,
):
    ...
```

⚠️ Cannot mix `Form`/`File` with JSON `Body(...)` — HTTP supports only one encoding per request.

## Form as a Pydantic model

```python
class Creds(BaseModel):
    username: str
    password: str

@app.post("/login/")
async def login(data: Annotated[Creds, Form()]): ...
```

## Gotchas

- ⚠️ Forgetting `python-multipart` → cryptic error at import/request time.
- ⚠️ Reading a large `UploadFile` via `await file.read()` may OOM — chunk instead.
