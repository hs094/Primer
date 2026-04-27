# Pydantic Knowledge Pack

Refresher vault mirroring [pydantic.dev/docs/validation/latest](https://pydantic.dev/docs/validation/latest/). Code-first, terse — for re-reading, not first-time learning.

🔑 **Why Pydantic:** type-hint-driven validation backed by a Rust core. `BaseModel` for structured data, `TypeAdapter` for ad-hoc types, `BaseSettings` for config, JSON Schema for free.

## Get Started

| # | Note | What it covers |
|---|---|---|
| 01 | [[Pydantic GS 01 - Welcome and Why]] | What Pydantic is and the case for using it |
| 02 | [[Pydantic GS 02 - Help with Pydantic]] | Where to get help (GitHub Discussions, Discord, SO tag) |
| 03 | [[Pydantic GS 03 - Installation]] | `pip install pydantic`, optional extras, `uv` |
| 04 | [[Pydantic GS 04 - Migration Guide]] | v1 → v2 breaking changes (`.dict`→`.model_dump`, etc.) |
| 05 | [[Pydantic GS 05 - Version Policy]] | Versioning, deprecation cadence, support windows |

## Concepts

| # | Note | What it covers |
|---|---|---|
| 01 | [[Pydantic CO 01 - Models]] | `BaseModel`, `model_validate`, `model_dump`, `RootModel`, dynamic models |
| 02 | [[Pydantic CO 02 - Fields]] | `Field()`, defaults, constraints, `Annotated[..., Field(...)]` |
| 03 | [[Pydantic CO 03 - JSON Schema]] | `model_json_schema()`, modes, `WithJsonSchema`, custom generators |
| 04 | [[Pydantic CO 04 - JSON]] | `model_validate_json`, `model_dump_json`, `Json[T]` field type |
| 05 | [[Pydantic CO 05 - Types]] | Built-in types, Pydantic types, `__get_pydantic_core_schema__` |
| 06 | [[Pydantic CO 06 - Unions]] | Smart vs left-to-right, `Discriminator`, tagged unions |
| 07 | [[Pydantic CO 07 - Alias]] | `alias` / `validation_alias` / `serialization_alias`, `AliasPath`/`AliasChoices` |
| 08 | [[Pydantic CO 08 - Configuration]] | `model_config = ConfigDict(...)` and the knobs |
| 09 | [[Pydantic CO 09 - Serialization]] | `model_dump`, `@field_serializer`, `@model_serializer`, `@computed_field` |
| 10 | [[Pydantic CO 10 - Validators]] | `@field_validator`, `@model_validator`, annotated forms |
| 11 | [[Pydantic CO 11 - Dataclasses]] | `pydantic.dataclasses.dataclass` and stdlib interop |
| 12 | [[Pydantic CO 12 - Forward Annotations]] | `model_rebuild()`, recursive models, string annotations |
| 13 | [[Pydantic CO 13 - Strict Mode]] | `Strict`, `StrictInt`, what coercion strict disables |
| 14 | [[Pydantic CO 14 - Type Adapter]] | `TypeAdapter` for non-`BaseModel` types |
| 15 | [[Pydantic CO 15 - Validation Decorator]] | `@validate_call`, `validate_return` |
| 16 | [[Pydantic CO 16 - Conversion Table]] | Type-coercion matrix (lax vs strict, JSON vs Python) |
| 17 | [[Pydantic CO 17 - Settings Management]] | `BaseSettings`, env loading, `.env`, secrets, CLI args |
| 18 | [[Pydantic CO 18 - Performance]] | `TypeAdapter` reuse, `model_dump_json`, `FailFast`, `model_construct` |
| 19 | [[Pydantic CO 19 - Experimental]] | Pipeline API, partial validation, `MISSING` |

## Examples

| # | Note | What it covers |
|---|---|---|
| 01 | [[Pydantic EX 01 - Validating File Data]] | Loading JSON / CSV / TOML / YAML through models |
| 02 | [[Pydantic EX 02 - Web and API Requests]] | FastAPI, requests, httpx — input/output validation |
| 03 | [[Pydantic EX 03 - Queues]] | Redis / RabbitMQ / Kafka payload validation |
| 04 | [[Pydantic EX 04 - Databases]] | SQLAlchemy interop via `from_attributes=True` |
| 05 | [[Pydantic EX 05 - Custom Validators]] | Phone, country, IBAN, postcode patterns |
| 06 | [[Pydantic EX 06 - Dynamic Models]] | `create_model`, runtime field generation |

## Error Messages

| # | Note | What it covers |
|---|---|---|
| 01 | [[Pydantic ER 01 - Error Handling]] | `ValidationError`, `.errors()`, `PydanticCustomError` |
| 02 | [[Pydantic ER 02 - Validation Errors]] | Catalog of validator error types (`int_parsing`, etc.) |
| 03 | [[Pydantic ER 03 - Usage Errors]] | `PydanticUserError` catalog (model misconfigurations) |

## Integrations

| # | Note | What it covers |
|---|---|---|
| 01 | [[Pydantic IN 01 - Logfire]] | `logfire.instrument_pydantic()` and validation tracing |
| 02 | [[Pydantic IN 02 - LLMs]] | `llms.txt`, `model_json_schema()` as structured-output schema |
| 03 | [[Pydantic IN 03 - Dev Tools]] | mypy plugin, VS Code, PyCharm, Hypothesis, datamodel-code-generator |

## Internals

| # | Note | What it covers |
|---|---|---|
| 01 | [[Pydantic IT 01 - Architecture]] | Rust core + Python layer, `__pydantic_core_schema__` |
| 02 | [[Pydantic IT 02 - Resolving Annotations]] | How string annotations resolve, `_types_namespace`, `defer_build` |

💡 **Takeaway:** start at [[Pydantic CO 01 - Models]] for `BaseModel`, [[Pydantic CO 17 - Settings Management]] for config, [[Pydantic CO 10 - Validators]] for custom logic, [[Pydantic CO 16 - Conversion Table]] when something coerces unexpectedly.
