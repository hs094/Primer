# HT 09 — Configure Swagger UI

🔑 Swagger UI accepts a JSON config blob. FastAPI surfaces it via `swagger_ui_parameters=` on the app constructor.

## Common tweaks

```python
from fastapi import FastAPI

app = FastAPI(
    swagger_ui_parameters={
        "syntaxHighlight": {"theme": "obsidian"},  # dark code blocks
        "deepLinking": True,                       # links to specific ops
        "displayRequestDuration": True,            # show ms for try-it-out
        "filter": True,                            # search filter box
        "tryItOutEnabled": True,                   # "Try it out" auto-enabled
        "defaultModelsExpandDepth": -1,            # hide the schemas section
        "docExpansion": "none",                    # collapse all by default
    }
)
```

## Disable syntax highlight

Heavy on huge schemas:

```python
swagger_ui_parameters={"syntaxHighlight": False}
```

## Persist authorisation across reloads

```python
swagger_ui_parameters={"persistAuthorization": True}
```

Useful when devs hit "Authorize" once and want it to stick across page refreshes.

## Initialise OAuth2

```python
app = FastAPI(
    swagger_ui_init_oauth={
        "clientId": "swagger-ui",
        "appName": "My API",
        "usePkceWithAuthorizationCodeGrant": True,
    }
)
```

## Full config reference

See [Swagger UI configuration](https://swagger.io/docs/open-source-tools/swagger-ui/usage/configuration/) — every key is a candidate for `swagger_ui_parameters`.

## Per-environment

```python
swagger_ui_parameters = (
    {"docExpansion": "list"} if settings.env == "dev"
    else {"docExpansion": "none", "filter": True}
)
app = FastAPI(swagger_ui_parameters=swagger_ui_parameters)
```

💡 Devs want everything expanded; reviewers / customers want collapsed and searchable. Configure per env.
