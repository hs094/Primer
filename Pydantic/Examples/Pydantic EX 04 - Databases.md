# EX 04 — Databases

🔑 **Key insight:** keep Pydantic models and SQLAlchemy models *separate*, bridge them with `from_attributes=True` and field aliases — fighting SQLAlchemy's reserved names is a config problem, not a metaclass problem.

## Why two models

ORM libraries map objects to database rows; Pydantic models validate and serialize. Conflating the two leads to either a watered-down ORM or a watered-down validator. The docs frame Pydantic as "a great tool for defining models for ORM (object relational mapping) libraries" — defining the **schema layer**, not the persistence layer.

If the duplication bothers you, the docs explicitly point at **SQLModel**, which "integrates Pydantic with SQLAlchemy such that much of the code duplication is eliminated." That is the right choice when one model genuinely should serve both purposes.

For pure Pydantic + SQLAlchemy, the canonical pattern keeps them as distinct classes and uses [[Pydantic CO 02 - Fields]] aliases for collisions.

## The aliasing pattern

`metadata` is reserved by SQLAlchemy's declarative base. Name the SQLAlchemy attribute `metadata_` and let Pydantic alias `metadata` → `metadata_`:

```python
import sqlalchemy as sa
from sqlalchemy.orm import declarative_base

from pydantic import BaseModel, ConfigDict, Field


class MyModel(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    metadata: dict[str, str] = Field(alias='metadata_')


Base = declarative_base()


class MyTableModel(Base):
    __tablename__ = 'my_table'
    id = sa.Column('id', sa.Integer, primary_key=True)
    # 'metadata' is reserved by SQLAlchemy, hence the '_'
    metadata_ = sa.Column('metadata', sa.JSON)


sql_model = MyTableModel(metadata_={'key': 'val'}, id=1)
pydantic_model = MyModel.model_validate(sql_model)

print(pydantic_model.model_dump())
#> {'metadata': {'key': 'val'}}
print(pydantic_model.model_dump(by_alias=True))
#> {'metadata_': {'key': 'val'}}
```

Three pieces are doing the work here:

1. **`from_attributes=True`** — tells Pydantic to pull values via `getattr(obj, name)` instead of `obj[name]`, so it can ingest an ORM instance directly. See [[Pydantic CO 01 - Models]] for `model_config` and [[Pydantic CO 14 - Type Adapter]] for the validation entry points.
2. **`Field(alias='metadata_')`** — Pydantic looks up `metadata_` on the SQLAlchemy object but exposes it as `metadata` on the Pydantic side.
3. **`by_alias=True`** on dump — round-trip back to the SQLAlchemy-shaped dict when you need to write through.

## Why aliases beat renames

You could rename the Pydantic field too, but then your API consumers see the SQLAlchemy oddity. Aliases keep the public schema clean (`metadata`) while accommodating the storage layer's quirks (`metadata_`). The docs note: "aliases have priority over field names for field population" — that is what makes the `model_validate(sql_model)` line work.

## Two flow directions

| Direction | Call |
|---|---|
| ORM row → Pydantic | `MyModel.model_validate(sql_row)` (needs `from_attributes=True`) |
| Pydantic → ORM kwargs | `MyTableModel(**model.model_dump(by_alias=True))` |

For wholesale dumps to JSON for an API response, drop `by_alias` so the consumer sees the clean field name.

⚠️ `from_attributes=True` will quietly walk relationships and lazy-load them. If your SQLAlchemy session has been closed, you will get a `DetachedInstanceError` mid-validation. Materialize relationships eagerly (`selectinload`, `joinedload`) before handing the row to Pydantic.

## Where this fits

- **API layer** — convert SQLAlchemy rows into Pydantic response models for FastAPI/serialization.
- **Write paths** — validate inbound payloads with Pydantic, then construct the SQLAlchemy model from the validated data.
- **Migration scripts** — Pydantic in the middle gives you a typed shape independent of the table definition.

For dataclass-shaped persistence helpers, see [[Pydantic CO 11 - Dataclasses]]. For deeper aliasing tricks (validation aliases vs serialization aliases, populate-by-name), see [[Pydantic CO 02 - Fields]].

💡 **Takeaway:** two models, one alias, `from_attributes=True` — that is the entire SQLAlchemy interop story; reach for SQLModel only when you genuinely want one class doing both jobs.
