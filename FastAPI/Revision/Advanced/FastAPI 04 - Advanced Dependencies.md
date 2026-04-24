# Adv 04 — Advanced Dependencies

## Parametric dependencies (callable-returning-callable)

```python
class FixedContentQueryChecker:
    def __init__(self, fixed_content: str):
        self.fixed_content = fixed_content

    def __call__(self, q: str = ""):
        return self.fixed_content in q

checker = FixedContentQueryChecker("bar")

@app.get("/query-checker")
async def read(matched: Annotated[bool, Depends(checker)]):
    return {"matched": matched}
```

- Objects with `__call__` are valid dependencies.
- Lets you parameterize behavior without a closure.

## Dependency factory

```python
def paginator(default_limit: int = 10):
    def inner(skip: int = 0, limit: int = default_limit):
        return {"skip": skip, "limit": limit}
    return inner

small_page = paginator(default_limit=20)

@app.get("/items/")
async def f(p: Annotated[dict, Depends(small_page)]):
    ...
```

## Use cache off

```python
Depends(expensive_dep, use_cache=False)
```

- By default, if the same dep appears multiple times in a single request, it's evaluated once. `use_cache=False` forces re-evaluation.

## Sub-dependency overrides for tests

```python
app.dependency_overrides[get_session] = lambda: mock_session
```

Remove: `app.dependency_overrides.pop(get_session)`.

## Dependencies with `yield` and exceptions (post-0.106)

Exceptions from the handler are **re-raised inside** the generator at the `yield` point. You can `try/except`:

```python
async def get_session():
    async with Session() as s:
        try:
            yield s
            await s.commit()
        except AppError:
            await s.rollback()
            raise
```

## Gotchas

- ⚠️ Don't raise `HTTPException` from teardown (after `yield`) — the response may already be sent. Use logging / alerting.
- ⚠️ Large dep graphs that re-call `get_session` with `use_cache=False` can exhaust the pool.
