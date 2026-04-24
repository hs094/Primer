# Adv 01 — Path Operation Advanced Config

## OpenAPI operationId

```python
@app.get("/items/", operation_id="list_items")
```

- Default is `function_name_path_method`.
- Client generators use this as the method name → stabilize it.

### Use function names as operationIds (one-liner)

```python
def use_route_names_as_operation_ids(app: FastAPI):
    for route in app.routes:
        if isinstance(route, APIRoute):
            route.operation_id = route.name
use_route_names_as_operation_ids(app)
```

Call **after** all routes are included.

## Exclude from OpenAPI

```python
@app.get("/internal/debug", include_in_schema=False)
```

Hidden from `/docs` and `/openapi.json` but still routed.

## Docstring magic

```python
@app.post("/items/")
async def create(item: Item):
    """
    Create an item.

    - **name**: each item has a name
    - **price**: required and > 0
    \f
    :param item: user input.
    """
```

- `\f` truncates the docstring for the OpenAPI description (useful for internal-only notes).
- Markdown supported.

## Extra response metadata

See [[03 - Additional Responses]].

## Override the default response class for a route

```python
from fastapi.responses import ORJSONResponse
@app.get("/", response_class=ORJSONResponse)
```

Or app-wide: `app = FastAPI(default_response_class=ORJSONResponse)`.

## Raw `openapi_extra`

```python
@app.post("/items/",
          openapi_extra={"x-internal": True,
                         "requestBody": {"content": {"application/x-yaml": {}}}})
```

For custom content types or vendor OpenAPI extensions. Overrides auto-generated schema.

## Accessing the request body manually (bypass auto-parse)

```python
@app.post("/items/", openapi_extra={...})
async def f(request: Request):
    raw = await request.body()
    ...
```

Useful when combined with `openapi_extra` to declare the schema yourself.
