# ER 03 — Usage Errors

🔑 **Key insight:** `PydanticUserError` fires when *your model definition* is wrong — not when end-user data is wrong — so every code in this catalogue is a static bug to fix at author time, never a runtime branch to handle.

Compare with [[Pydantic ER 01 - Error Handling]] (`ValidationError` for bad input) and [[Pydantic ER 02 - Validation Errors]] (the runtime data-error catalogue).

## Annotation & forward-ref problems

| Code | Trigger |
|---|---|
| `class-not-fully-defined` | A type referenced in an annotation isn't defined or is defined after first use — see [[Pydantic CO 12 - Forward Annotations]] |
| `undefined-annotation` | Undefined annotation hit during CoreSchema generation, e.g. `a: 'B'` where `B` never exists; surfaces on `model_rebuild()` |
| `unevaluable-type-annotation` | Type-annotation name clashes with a field name (e.g. field `date: date` where the import is shadowed) |
| `schema-for-unknown-type` | Pydantic can't generate a schema for the given type, e.g. `x: 43 = 123` |
| `model-field-missing-annotation` | Field has no type annotation — `a = Field('foobar')` or `b = None` with no hint |
| `invalid-annotated-type` | An `Annotated[...]` constraint can't apply to its target — `foo: Annotated[str, FutureDate()]` |
| `circular-reference-schema` | Cyclic type definition causes infinite schema recursion — `type A = A` |
| `invalid-self-type` | `Self` used outside a valid context (e.g. `@validate_call def f(self: Self):`) |

## Decorator misconfiguration (see [[Pydantic CO 10 - Validators]])

| Code | Trigger |
|---|---|
| `decorator-missing-arguments` | `@field_validator` / `@field_serializer` used with no field args |
| `decorator-invalid-fields` | Field args aren't strings — `@field_validator(['a','b'])` |
| `decorator-missing-field` | Decorator references a field the model doesn't have |
| `validator-instance-method` | Validator on instance method instead of `@classmethod` |
| `validator-input-type` | `json_schema_input_type` used with a mode other than `before`/`plain`/`wrap` |
| `validator-field-config-info` | Deprecated `field` / `config` parameters in validator signature |
| `validator-v1-signature` | V1-style validator signature no longer supported |
| `validator-signature` | Wrong signature for `field_validator` / `model_validator` |
| `field-serializer-signature` | `@field_serializer` function has wrong signature |
| `model-serializer-signature` | `@model_serializer` function has wrong signature |
| `model-serializer-instance-method` | `@model_serializer` on a `@classmethod` or missing `self` |
| `multiple-field-serializers` | Two `@field_serializer` decorators on the same field |
| `root-validator-pre-skip` | `@root_validator` with `skip_on_failure=False` |

## Discriminated unions (see [[Pydantic CO 06 - Unions]])

| Code | Trigger |
|---|---|
| `discriminator-no-field` | A union member lacks the discriminator field |
| `discriminator-needs-literal` | Discriminator field isn't typed as `Literal[...]` |
| `discriminator-alias-type` | Non-string alias on the discriminator |
| `discriminator-alias` | Different aliases on the discriminator across union members |
| `discriminator-validator` | `before` / `wrap` / `plain` validator placed on a discriminator |
| `callable-discriminator-no-tag` | Callable `Discriminator` without `Tag` annotations on each branch |

## Configuration conflicts (see [[Pydantic CO 08 - Configuration]])

| Code | Trigger |
|---|---|
| `config-both` | Both `class Config` and `model_config` declared on the same model |
| `model-field-overridden` | Subclass overrides a base-class field with an *unannotated* attribute |
| `model-config-invalid-field-name` | A field is literally named `model_config` |
| `with-config-on-model` | `@with_config` placed on an existing `BaseModel` subclass |
| `dataclass-on-model` | `@dataclass` placed on a `BaseModel` subclass |
| `dataclass-init-false-extra-allow` | Dataclass with `extra='allow'` and an `init=False` field |
| `clashing-init-and-init-var` | Field with both `init=False` and `init_var=True` |
| `root-model-extra` | `extra` config set on a `RootModel` subclass |
| `validate-by-alias-and-name-false` | Both `validate_by_alias=False` and `validate_by_name=False` |
| `type-adapter-config-unused` | Passed `config` to [[Pydantic CO 14 - Type Adapter|`TypeAdapter`]] for a type that has its own config |

## API misuse

| Code | Trigger |
|---|---|
| `base-model-instantiated` | Calling `BaseModel()` directly instead of subclassing |
| `create-model-field-definitions` | Bad field-definition tuple — `create_model('M', foo=(str, 'd', 'extra'))` |
| `removed-kwargs` | V1 keyword passed to V2 — e.g. `Field(regex=...)` |
| `import-error` | Importing a name removed in V2 |
| `custom-json-schema` | Defining the deprecated `__modify_schema__` instead of `__get_pydantic_json_schema__` |
| `invalid-for-json-schema` | JSON-schema generation fails for a CoreSchema (e.g. `ImportString` field) |
| `json-schema-already-used` | Reusing a `GenerateJsonSchema` instance after it has produced a schema |
| `validate-call-type` | `@validate_call` applied to an unsupported callable (class, bare classmethod, instance with no `self`) |

## Typing-extensions & TypedDict

| Code | Trigger |
|---|---|
| `typed-dict-version` | Used `typing.TypedDict` instead of `typing_extensions.TypedDict` on Python < 3.12 |
| `unpack-typed-dict` | `Unpack[...]` used on `**kwargs` with a non-TypedDict |
| `overlapping-unpack-typed-dict` | Unpacked TypedDict has a field name colliding with a normal parameter |

⚠️ A `PydanticUserError` at import time means the module never finishes loading. Don't try/except around it — fix the model definition.

⚠️ `class-not-fully-defined` and `undefined-annotation` are the two you'll hit most often when forward refs aren't resolved; the fix is almost always `Model.model_rebuild()` once the missing name is in scope. See [[Pydantic IT 02 - Resolving Annotations]].

💡 **Takeaway:** If pydantic raises `PydanticUserError`, treat it as a compile-time signal — change the model, not your error handler.
