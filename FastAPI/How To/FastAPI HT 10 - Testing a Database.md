# HT 10 — Testing a Database

🔑 Hit a real DB in tests, isolated per test. Don't mock the ORM — mocks pass while migrations break in prod.

## Two viable strategies

### A) Transaction-per-test rollback (fast, single DB)

```python
import pytest, pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

@pytest_asyncio.fixture
async def session():
    engine = create_async_engine(TEST_DSN)
    async with engine.connect() as conn:
        trans = await conn.begin()
        Session = async_sessionmaker(bind=conn, expire_on_commit=False)
        async with Session() as s:
            yield s
        await trans.rollback()
    await engine.dispose()
```

Override the app dep:

```python
@pytest_asyncio.fixture
async def client(session):
    async def override(): yield session
    app.dependency_overrides[get_session] = override
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

### B) Disposable DB per test class (slower, full isolation)

Spin a fresh schema (e.g. with `testcontainers`):

```python
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def pg():
    with PostgresContainer("postgres:16") as p:
        yield p.get_connection_url()
```

Run Alembic migrations on top, share for the session.

## Migrations in tests

Don't `Base.metadata.create_all()` — it skips Alembic and your test DB will diverge from prod schema. Run migrations:

```python
from alembic.config import Config
from alembic import command
cfg = Config("alembic.ini")
cfg.set_main_option("sqlalchemy.url", TEST_DSN)
command.upgrade(cfg, "head")
```

## What to assert

- Behavior at the API boundary (`POST /items`, then `GET /items/{id}`).
- Constraint violations bubble up as proper errors.
- Concurrent paths (use `pytest-xdist`) don't deadlock.

⚠️ Don't share state across tests via module globals; rely on the fixture rollback or fresh-DB setup.

💡 Fast test feedback loop > "100% mocked". Shoot for sub-second per test against a real Postgres with rollback.

See [[FastAPI 21 - Testing Dependency Overrides]] and [[FastAPI 24 - Testing and Debugging]].
