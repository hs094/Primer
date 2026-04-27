# Adv 26 — JSON with Bytes as Base64

🔑 JSON has no native binary type. Pydantic v2 can encode/decode `bytes` fields as **base64** in JSON via `model_config` flags — both for incoming bodies and outgoing responses.

## Input — `val_json_bytes`

```python
from fastapi import FastAPI
from pydantic import BaseModel

class DataInput(BaseModel):
    description: str
    data: bytes
    model_config = {"val_json_bytes": "base64"}

app = FastAPI()

@app.post("/data")
async def post_data(body: DataInput):
    content = body.data.decode("utf-8")
    return {"description": body.description, "content": content}
```

Client sends:

```json
{"description": "Some data", "data": "aGVsbG8="}
```

`"aGVsbG8="` is base64 for `hello`. By the time `post_data` runs, `body.data` is plain `bytes`.

## Output — `ser_json_bytes`

```python
class DataOutput(BaseModel):
    description: str
    data: bytes
    model_config = {"ser_json_bytes": "base64"}

@app.get("/data")
async def get_data() -> DataOutput:
    data = "hello".encode("utf-8")
    return DataOutput(description="A plumbus", data=data)
```

Response body has `"data": "aGVsbG8="`.

## Both directions

```python
class DataInputOutput(BaseModel):
    description: str
    data: bytes
    model_config = {
        "val_json_bytes": "base64",
        "ser_json_bytes": "base64",
    }

@app.post("/data-in-out")
async def post_data_in_out(body: DataInputOutput) -> DataInputOutput:
    return body
```

## When to prefer multipart

If files are large (>1 MB) or already binary on the client (browser `File` object), use [[FastAPI 14 - Forms and Files]] (`UploadFile`) — multipart streams avoid the ~33 % base64 overhead and the full-buffer-in-RAM trap.

⚠️ A 10 MB binary becomes ~13.3 MB of JSON — and that's *plus* the encode/decode CPU. If you're shoveling bytes regularly, use a separate binary endpoint or signed object-storage URLs.

💡 Base64 is fine for ≤ a few hundred KB, easily inlined in JSON. Beyond that, multipart wins.
