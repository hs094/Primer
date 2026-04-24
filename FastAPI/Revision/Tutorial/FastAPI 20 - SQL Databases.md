# 20 — SQL Databases

Two common paths: **SQLModel** (Tiangolo's lib, Pydantic + SQLAlchemy) or **SQLAlchemy 2.x async** directly.

## SQLModel basics

```python
from sqlmodel import Field, SQLModel, create_engine, Session, select

class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    age: int | None = None

engine = create_engine("sqlite:///db.sqlite", connect_args={"check_same_thread": False})
SQLModel.metadata.create_all(engine)

def get_session():
    with Session(engine) as session:
        yield session

@app.post("/heroes/")
def create(hero: Hero, session: Annotated[Session, Depends(get_session)]):
    session.add(hero); session.commit(); session.refresh(hero)
    return hero

@app.get("/heroes/", response_model=list[Hero])
def list_heroes(session: Annotated[Session, Depends(get_session)]):
    return session.exec(select(Hero)).all()
```

One class doubles as DB table and response model (opinionated, fast to ship).

## Async SQLAlchemy 2.x (preferred for prod)

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from collections.abc import AsyncIterator

engine = create_async_engine(settings.database_url, echo=False, pool_size=10)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)

class Base(DeclarativeBase): ...

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

async def get_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

Handler:

```python
@app.post("/users/", response_model=UserOut)
async def create(payload: UserIn, session: Annotated[AsyncSession, Depends(get_session)]):
    user = User(name=payload.name)
    session.add(user)
    await session.flush()
    return user
```

## Separate in / out / DB models

- `UserIn` — client-supplied, includes password.
- `UserOut` — `id`, `name`, `email` — no secrets.
- `User` (DB) — storage representation.

Avoids leakage and keeps schemas honest.

## Migrations

Use **Alembic**. Generate revisions with `alembic revision --autogenerate`, apply with `alembic upgrade head`. Use SQLAlchemy API in migrations — not raw SQL — for engine portability.

## Connection pooling

```python
create_async_engine(url, pool_size=10, max_overflow=20, pool_pre_ping=True)
```

- `pool_pre_ping` avoids stale connections across proxy restarts.
- Match `pool_size * workers` against your DB's `max_connections`.

## Gotchas

- ⚠️ `expire_on_commit=True` (default) triggers lazy reloads after `commit()` — in async this raises `MissingGreenlet`. Set `expire_on_commit=False` on your async sessionmaker.
- ⚠️ Don't hold a session across background tasks; open a new one.
- ⚠️ One session per request (via `Depends`), not a module-level singleton.
