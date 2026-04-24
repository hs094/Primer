# Adv 10 — Generate Clients

🔑 FastAPI's OpenAPI schema drives frontend SDK generation for free.

## Export the schema

```bash
curl http://localhost:8000/openapi.json > openapi.json
# or at build time:
python -c "from main import app, json; \
           open('openapi.json','w').write(json.dumps(app.openapi()))"
```

## TypeScript clients

Popular tools:

- `@hey-api/openapi-ts` (modern, actively maintained) — recommended
- `openapi-typescript-codegen`
- `openapi-generator-cli` (polyglot, heavier)

```bash
npx @hey-api/openapi-ts \
  -i ./openapi.json \
  -o ./src/client \
  -c @hey-api/client-fetch
```

## Stable operation IDs → good method names

Without help, FastAPI generates ugly IDs like `read_item_items__item_id__get`. Rename all to match the function name:

```python
from fastapi.routing import APIRoute

def simplify_ids(app: FastAPI):
    for route in app.routes:
        if isinstance(route, APIRoute):
            route.operation_id = route.name   # function name

simplify_ids(app)
```

Do this **after** including all routers.

## Custom client generation tags

`tags=[...]` on operations → grouped into separate SDK "modules" / classes.

## Model reuse

- Pydantic models → TypeScript interfaces with the same names.
- Name them well: `UserCreate`, `UserOut` beat `UserIn1`, `UserIn2`.
- Use inheritance / `Literal` for enums; they translate cleanly.

## Gotchas

- ⚠️ Regenerate the client whenever the API changes; CI step is cheap insurance.
- ⚠️ Don't hand-edit generated code — gets overwritten.
- ⚠️ Enum values are strings on the wire; keep `str, Enum` for stability.
