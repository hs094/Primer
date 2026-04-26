# Adv 16 — Response: Change Status Code

🔑 Inject the `Response` object as a parameter to mutate status / headers / cookies *while* still returning a normal Python value (FastAPI keeps doing serialization).

## Pattern

```python
from fastapi import FastAPI, Response, status

app = FastAPI()
tasks = {"foo": "Foo task"}

@app.put("/tasks/{name}")
async def upsert(name: str, response: Response):
    if name not in tasks:
        response.status_code = status.HTTP_201_CREATED
    tasks[name] = name
    return {"name": tasks[name]}
```

- Default decorator status (e.g. 200) used unless you assign `response.status_code`.
- Body still goes through `response_model` filtering / encoder.

## Vs returning a `Response` directly

| Need | Approach |
|---|---|
| Different status, *same* serialization | Inject `Response`, set `.status_code` |
| Different body shape / bytes | Return `JSONResponse(...)` ([[FastAPI 14 - Additional Status Codes]]) |
| Errors | `raise HTTPException(...)` ([[FastAPI 15 - Error Handling]]) |

## Set headers / cookies the same way

```python
async def f(response: Response):
    response.headers["X-Cache"] = "MISS"
    response.set_cookie("seen", "1")
    return ...
```

⚠️ Don't set the body on the injected `Response` — it'll be ignored. Return data normally.

💡 Reach for this when *most* responses have the default status and only one branch differs.
