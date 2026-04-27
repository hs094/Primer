# IN 03 — Dev Tools

🔑 **Key insight:** Pydantic ships first-party plugins or guidance for every major Python dev tool — mypy, Pylance/VS Code, PyCharm, Hypothesis, datamodel-code-generator — so you can keep your IDE, type checker, and tests in sync with model behaviour without writing your own glue.

## mypy plugin

Enable in your mypy config; for V1 models use `pydantic.v1.mypy` instead.

```ini
[mypy]
plugins = pydantic.mypy
```

```toml
[tool.mypy]
plugins = ["pydantic.mypy"]
```

The plugin synthesises typed `__init__` signatures (required fields become required kwargs), gives `model_construct` accurate types, detects mutations on frozen models, validates that `default` / `default_factory` values match field types, flags un-annotated fields, and warns on required dynamic aliases that defeat type checking.

Tunables go under their own section:

```ini
[mypy]
plugins = pydantic.mypy
disallow_untyped_defs = True

[pydantic-mypy]
init_forbid_extra = True
init_typed = True
warn_required_dynamic_aliases = True
```

```toml
[tool.mypy]
plugins = ["pydantic.mypy"]
disallow_untyped_defs = true

[tool.pydantic-mypy]
init_forbid_extra = true
init_typed = true
warn_required_dynamic_aliases = true
```

| Option | Effect |
|---|---|
| `init_typed` | Use real field types in `__init__` (not `Any`); accounts for Pydantic's coercion |
| `init_forbid_extra` | Drop the `**kwargs: Any` from synthesised `__init__`, rejecting unknown args |
| `warn_required_dynamic_aliases` | Error when dynamic aliases exist on models without `validate_by_name` |

## Visual Studio Code (Pylance / Pyright)

Pylance uses Pyright under the hood; with type checking enabled you get IntelliSense, autocomplete, and required-arg checks on every `Model(...)` call.

Setup:

1. Install Pylance (ships with the Python extension; verify it's enabled).
2. Select the right interpreter / venv (the one Pydantic is installed in).
3. Set `Python › Analysis: Type Checking Mode` to `basic` or `strict` (default is `off`).
4. *Optional:* enable mypy via `Python › Linting: Mypy Enabled` to surface the [[#mypy plugin|pydantic.mypy]] diagnostics inline.

The docs note: *"Pylance/pyright requires `default` to be a keyword argument to `Field` in order to infer that the field is optional."*

```python
from pydantic import BaseModel, Field

class Knight(BaseModel):
    title: str = Field(default='Sir Lancelot')  # Correct
    age: int = Field(23)                        # Problematic for Pylance
```

Frozen models are detected and mutations are flagged:

```python
from pydantic import BaseModel

class Knight(BaseModel, frozen=True):
    title: str
    age: int
    color: str = 'blue'
```

Suppressing false positives where Pydantic *will* coerce (e.g. `'23'` → `23`):

```python
lancelot = Knight(title='Sir Lancelot', age='23')  # pyright: ignore
lancelot = Knight(title='Sir Lancelot', age='23')  # type: ignore
```

Or escape-hatch via `Any` / `cast`:

```python
from typing import Any
from pydantic import BaseModel

class Knight(BaseModel):
    title: str
    age: int
    color: str = 'blue'

age_str: Any = '23'
lancelot = Knight(title='Sir Lancelot', age=age_str)
```

```python
from typing import Any, cast
from pydantic import BaseModel

class Knight(BaseModel):
    title: str
    age: int
    color: str = 'blue'

lancelot = Knight(title='Sir Lancelot', age=cast(Any, '23'))
```

⚠️ Pydantic is intentionally lenient (`'23'` becomes `23`); strict type checkers will flag those calls as errors. The docs explicitly call this out as a possible source of false positives — pick `basic` mode or use `# type: ignore` rather than abandoning the checker.

## PyCharm plugin

Free, installed via **Preferences → Plugin → Marketplace → "pydantic"**. It adds three things to `BaseModel.__init__`:

- inspection of init parameters
- autocompletion of `__init__` arguments
- type-checking validation against field annotations

It also handles refactoring: renaming a field updates `__init__` calls in parents and children, and renaming an `__init__` keyword updates the underlying field declaration. No configuration needed beyond installation.

Plugin home: <https://plugins.jetbrains.com/plugin/12861-pydantic> — source: <https://github.com/koxudaxi/pydantic-pycharm-plugin>.

## Hypothesis

⚠️ **Removed in Pydantic V2.** The official docs state:

> "Pydantic v2.0 drops built-in support for Hypothesis and no more ships with the integrated Hypothesis plugin."

> "We are temporarily removing the Hypothesis plugin in favor of studying a different mechanism."

The Pydantic team is working through a replacement (tracked in <https://github.com/pydantic/pydantic/issues/4682>), tied to discussion in the `annotated-types` project. Until then, write Hypothesis strategies by hand against your model fields — there is no `register_type_strategy` shipped from Pydantic v2.

## datamodel-code-generator

Generates Pydantic models from OpenAPI 3, JSON Schema, JSON/YAML/CSV data, Python dicts, and GraphQL schemas.

```bash
pip install datamodel-code-generator
```

Generate a model from a JSON Schema:

```bash
datamodel-codegen --input person.json --input-file-type jsonschema --output model.py
```

A typical generated output:

```python
from pydantic import BaseModel, Field, conint

class Pet(BaseModel):
    name: str | None = None
    age: int | None = None

class Person(BaseModel):
    first_name: str = Field(description="The person's first name.")
    last_name: str = Field(description="The person's last name.")
    age: conint(ge=0) | None = Field(None, description='Age in years.')
    pets: list[Pet] | None = None
    comment: Any | None = None
```

The tool preserves descriptions as field documentation and translates schema constraints into [[Pydantic CO 02 - Fields|`Field`]] / `conint` annotations. Full reference: <https://koxudaxi.github.io/datamodel-code-generator/>.

⚠️ Treat the generated `model.py` as source you own — re-running the generator will overwrite manual edits. Either commit the output and regenerate deliberately, or extend via subclassing in a separate module.

💡 **Takeaway:** Turn on `pydantic.mypy` + Pylance type-checking and you'll catch most Pydantic mistakes statically; reach for datamodel-code-generator when you have an existing schema to mirror, and write Hypothesis strategies by hand until V2 ships its replacement.
