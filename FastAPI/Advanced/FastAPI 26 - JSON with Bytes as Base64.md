# Adv 26 — JSON with Bytes as Base64

🔑 JSON has no native binary type. Pydantic encodes `bytes` fields as **base64** in JSON — both for incoming bodies and outgoing responses.

## Field type

```python
from pydantic import BaseModel, Base64Bytes, Base64UrlBytes

class FileUp(BaseModel):
    name: str
    payload: Base64Bytes        # standard base64
    token: Base64UrlBytes       # URL-safe variant
```

`Base64Bytes` (and friends) accept a base64 string in JSON, decode to `bytes` for your handler, and re-encode on the way out.

## Plain `bytes`

```python
class Doc(BaseModel):
    blob: bytes
```

Pydantic v2 accepts only base64-encoded strings here too. Plain non-base64 bytes in JSON would be invalid.

## In a route

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/upload")
async def upload(f: FileUp):
    # f.payload is already decoded bytes
    save(f.name, f.payload)
    return {"size": len(f.payload)}
```

Client sends:

```json
{"name": "x.bin", "payload": "SGVsbG8="}
```

## When to prefer multipart

If files are large (>1 MB) or already binary on the client (browser `File` object), use [[FastAPI 14 - Forms and Files]] (`UploadFile`) — multipart streams avoid the ~33 % base64 overhead and the full-buffer-in-RAM trap.

## Encoding outbound responses

Same model the other way:

```python
@app.get("/doc/{id}", response_model=Doc)
async def get(id: int) -> Doc:
    return Doc(blob=open("file.bin", "rb").read())
```

Response body has `"blob": "<base64>"`.

⚠️ A 10 MB binary becomes ~13.3 MB of JSON — and that's *plus* the encode/decode CPU. If you're shoveling bytes regularly, use a separate binary endpoint.

💡 Base64 is fine for ≤ a few hundred KB, easily inlined in JSON. Beyond that, multipart or signed object-storage URLs win.
