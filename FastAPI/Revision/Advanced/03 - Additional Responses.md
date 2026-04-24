# Adv 03 — Additional Responses, Headers, Cookies

## Document other responses

```python
@app.get("/items/{id}",
         response_model=Item,
         responses={
             404: {"model": Error, "description": "Not found"},
             500: {"description": "Server error",
                   "content": {"application/json": {"example": {"code": "oops"}}}},
         })
```

- Main response is `200` via `response_model`.
- Additional ones surface in docs + OpenAPI but aren't enforced at runtime — you still have to return or raise them.

## Reusable responses dict

```python
common = {404: {"model": Error}, 401: {"model": Error}}

@app.get("/a", responses={**common, 403: {"model": Error}})
```

## Mutating the response object

```python
from fastapi import Response

@app.post("/items/")
def create(response: Response):
    response.headers["X-Custom"] = "1"
    response.set_cookie("sid", "abc", httponly=True, secure=True, samesite="lax")
    response.status_code = status.HTTP_202_ACCEPTED
    return {"queued": True}
```

- Inject `Response` — mutating it propagates to the final response.
- Works alongside `response_model` (unlike returning a `Response` directly).

## Override headers / cookies in returned Response

```python
return JSONResponse(content={...}, headers={"X-Custom": "1"})
resp = JSONResponse({...})
resp.set_cookie("k", "v")
return resp
```

## `response_model` + `status_code` reminder

See [[Tutorial/13 - Extra Models and Status Codes]].

## Gotchas

- ⚠️ `responses={}` is docs-only; your handler still must actually produce those statuses.
- ⚠️ Cookies set via `response.set_cookie` persist only on that single response; subsequent requests need their own set.
