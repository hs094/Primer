# 09 — Declare Examples

🔑 Examples improve `/docs` UX + are surfaced in the OpenAPI schema for client generators.

## Pydantic model-level examples

```python
class Item(BaseModel):
    name: str
    price: float

    model_config = {
        "json_schema_extra": {
            "examples": [
                {"name": "Foo", "price": 35.4},
            ]
        }
    }
```

## `Field()` examples

```python
price: float = Field(examples=[3.14, 5.9])
```

## Parameter / body examples

```python
item: Annotated[Item, Body(examples=[{"name": "Foo", "price": 42}])]
```

## OpenAPI named examples (Swagger picks them from a dropdown)

```python
item: Annotated[
    Item,
    Body(openapi_examples={
        "normal":  {"summary": "Normal", "value": {"name": "Foo", "price": 35.4}},
        "invalid": {"summary": "Invalid", "value": {"name": "Foo", "price": "oops"}},
    }),
]
```

## Precedence

`openapi_examples` (named, rich) → `examples=[...]` (positional list) → `json_schema_extra`.

## Gotcha

- ⚠️ `example=` (singular, Pydantic v1) is deprecated; use `examples=[...]`.
