# HT 03 — GraphQL

🔑 FastAPI doesn't ship GraphQL itself, but mounts any GraphQL ASGI app cleanly. The docs list four options: **Strawberry** (recommended — typed, closest to FastAPI's design), **Ariadne** (schema-first), **Tartiflette** (via `tartiflette-asgi`), and **Graphene** (via `starlette-graphene3`).

## Strawberry

```bash
pip install "strawberry-graphql[fastapi]"
```

```python
import strawberry
from strawberry.fastapi import GraphQLRouter
from fastapi import FastAPI

@strawberry.type
class User:
    name: str
    age: int

@strawberry.type
class Query:
    @strawberry.field
    def user(self) -> User:
        return User(name="Patrick", age=100)

schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema)

app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")
```

`/graphql` → both POST endpoint and the playground UI.

## Sharing context (auth, DB)

```python
async def get_context(token: str = Depends(oauth2_scheme),
                      db: AsyncSession = Depends(get_session)):
    return {"user": await user_from(token, db), "db": db}

graphql_app = GraphQLRouter(schema, context_getter=get_context)
```

Inside resolvers: `info.context["user"]`.

## Ariadne (schema-first)

```python
from ariadne import QueryType, make_executable_schema, gql
from ariadne.asgi import GraphQL
from fastapi import FastAPI

type_defs = gql("type Query { hello: String! }")
query = QueryType()

@query.field("hello")
def resolve_hello(*_): return "world"

schema = make_executable_schema(type_defs, query)
app = FastAPI()
app.mount("/graphql", GraphQL(schema, debug=True))
```

## When GraphQL?

- ✅ Mobile / front-ends with wildly different field needs.
- ✅ Multiple consumers needing flexible projections.
- ⚠️ Costs: N+1 queries, complex caching, harder rate limiting, weaker OpenAPI tooling.
- ❌ Don't pick GraphQL for "future-proofing" a single client.

💡 REST + judicious `?fields=` and a sane response model often beats GraphQL for one consumer.
