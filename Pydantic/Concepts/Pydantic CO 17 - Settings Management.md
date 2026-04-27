# CO 17 — Settings Management

🔑 **Key insight:** `pydantic-settings` turns a `BaseSettings` subclass into a typed, multi-source config — env, `.env`, secrets dirs, CLI, cloud vaults — with a clear priority order and a single override hook.

## Install

```bash
pip install pydantic-settings
```

It ships separately from `pydantic` itself.

## Minimal BaseSettings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    auth_key: str = 'default'
    api_key: str = 'default'

    model_config = SettingsConfigDict(env_prefix='MY_')
```

Reads `MY_AUTH_KEY` / `MY_API_KEY` from the environment. Env match is **case-insensitive by default**.

## Field aliases for env names

```python
from pydantic import Field, AliasChoices
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    auth_key: str = Field(validation_alias='my_auth_key')
    api_key: str = Field(alias='my_api_key')
    redis_dsn: str = Field(
        'redis://user:pass@localhost:6379/1',
        validation_alias=AliasChoices('service_redis_dsn', 'redis_url')
    )
```

`AliasChoices` walks the list and uses the first hit. See [[Pydantic CO 07 - Alias]].

## Common config keys

| Key | Purpose |
|---|---|
| `env_prefix` | Prefix on every env var name |
| `env_nested_delimiter` | Splits env keys into nested fields |
| `env_nested_max_split` | Caps nesting depth |
| `case_sensitive` | Toggle case-sensitive env match |
| `env_ignore_empty` | Skip empty env values |
| `env_file` | Path (or tuple) to `.env` |
| `env_file_encoding` | Encoding for `.env` |
| `secrets_dir` | Directory of secret files |
| `dotenv_filtering` | `'match_prefix'` or `'only_existing'` |
| `validate_default` | Validate field defaults |
| `cli_parse_args` | Enable CLI parsing |

## Nested env vars

```python
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict

class DeepSubModel(BaseModel):
    v4: str

class SubModel(BaseModel):
    v1: str
    v2: bytes
    v3: int
    deep: DeepSubModel

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_nested_delimiter='__')
    sub_model: SubModel
```

```env
SUB_MODEL__V1=json-1
SUB_MODEL__V2=nested-2
SUB_MODEL__DEEP__V4=v4
```

## .env file loading

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file='.env',
        env_file_encoding='utf-8',
    )
```

Multiple files (later wins):

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=('.env', '.env.prod'))
```

Override at instantiation:

```python
settings = Settings(_env_file='prod.env', _env_file_encoding='utf-8')
```

`_env_file=None` disables.

## Disabling JSON parsing per field

```python
from typing import Annotated
from pydantic import field_validator
from pydantic_settings import BaseSettings, NoDecode

class Settings(BaseSettings):
    numbers: Annotated[list[int], NoDecode]

    @field_validator('numbers', mode='before')
    @classmethod
    def decode_numbers(cls, v: str) -> list[int]:
        return [int(x) for x in v.split(',')]
```

Use `ForceDecode` for the inverse when `enable_decoding=False` is set globally.

## Secrets directory

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(secrets_dir='/var/run')
    database_password: str
```

The file `/var/run/database_password` provides the value. Multi-dir tuple supported; later overrides earlier.

### Nested secrets

```python
from pydantic import BaseModel, SecretStr
from pydantic_settings import (
    BaseSettings,
    NestedSecretsSettingsSource,
    SettingsConfigDict,
)

class DbSettings(BaseModel):
    user: str
    passwd: SecretStr

class Settings(BaseSettings):
    db: DbSettings

    model_config = SettingsConfigDict(
        secrets_dir='secrets',
        secrets_nested_delimiter='_',
        env_prefix='MY_',
    )

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls,
        init_settings,
        env_settings,
        dotenv_settings,
        file_secret_settings,
    ):
        return (
            init_settings,
            env_settings,
            dotenv_settings,
            NestedSecretsSettingsSource(file_secret_settings),
        )
```

### Docker secrets

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(secrets_dir='/run/secrets')
    my_secret_data: str
```

```bash
printf "This is a secret" | docker secret create my_secret_data -
docker service create --name pydantic-with-secrets \
  --secret my_secret_data pydantic-app:latest
```

## Source priority (default)

Highest → lowest:

1. **Init args** (`Settings(foo='x')`)
2. **CLI args** (when `cli_parse_args=True`)
3. **Environment variables**
4. **Dotenv file**
5. **Secrets files**
6. **Field defaults**

## Customising sources

Override `settings_customise_sources` to reorder, add, or drop sources.

```python
from pydantic_settings import (
    BaseSettings,
    PydanticBaseSettingsSource,
    CliSettingsSource,
)

class Settings(BaseSettings):
    my_setting: str = 'default'

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (
            env_settings,
            init_settings,
            CliSettingsSource(settings_cls, cli_parse_args=True),
            dotenv_settings,
            file_secret_settings,
        )
```

## CLI parsing

```python
import sys
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict

class DeepSubModel(BaseModel):
    v4: str

class SubModel(BaseModel):
    v1: str
    v2: bytes
    v3: int
    deep: DeepSubModel

class Settings(BaseSettings):
    model_config = SettingsConfigDict(cli_parse_args=True)
    v0: str
    sub_model: SubModel

sys.argv = [
    'example.py',
    '--v0=0',
    '--sub_model={"v1": "json-1", "v2": "json-2"}',
    '--sub_model.v2=nested-2',
    '--sub_model.v3=3',
    '--sub_model.deep.v4=v4',
]
print(Settings().model_dump())
```

CLI list styles:

```python
class Settings(BaseSettings, cli_parse_args=True):
    my_list: list[int]

# JSON:    --my_list [1,2]
# argparse: --my_list 1 --my_list 2
# lazy:     --my_list 1,2
```

### CliApp.run

```python
from pydantic_settings import BaseSettings, CliApp

class Settings(BaseSettings):
    this_foo: str

    def cli_cmd(self) -> None:
        print(self.model_dump())
        self.this_foo = 'ran the foo cli cmd'

s = CliApp.run(Settings, cli_args=['--this_foo', 'is such a foo'])
print(s.model_dump())
```

`CliApp` defaults for `BaseModel`/dataclass targets are sane: `case_sensitive=True`, `cli_hide_none_type=True`, `cli_avoid_json=True`, `cli_enforce_required=True`, `cli_implicit_flags=True`, `cli_kebab_case=True`, `nested_model_default_partial_update=True`.

### Subcommands

```python
from pydantic import BaseModel
from pydantic_settings import (
    BaseSettings, CliPositionalArg, CliSubCommand, get_subcommand,
)

class Init(BaseModel):
    directory: CliPositionalArg[str]

class Clone(BaseModel):
    repository: CliPositionalArg[str]
    directory: CliPositionalArg[str]

class Git(BaseSettings, cli_parse_args=True, cli_exit_on_error=False):
    clone: CliSubCommand[Clone]
    init: CliSubCommand[Init]

cmd = Git()
subcommand = get_subcommand(cmd)
```

### Async cli_cmd

```python
from pydantic_settings import BaseSettings, CliApp

class AsyncSettings(BaseSettings):
    async def cli_cmd(self) -> None:
        print('Hello from async CLI method!')

CliApp.run(AsyncSettings, cli_args=[])
```

## AWS Secrets Manager

```python
import os
from pydantic import BaseModel
from pydantic_settings import (
    AWSSecretsManagerSettingsSource,
    BaseSettings,
    PydanticBaseSettingsSource,
)

class SubModel(BaseModel):
    a: str

class AWSSecretsManagerSettings(BaseSettings):
    foo: str
    bar: int
    sub: SubModel

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        aws_settings = AWSSecretsManagerSettingsSource(
            settings_cls,
            os.environ['AWS_SECRETS_MANAGER_SECRET_ID'],
        )
        return (
            init_settings,
            env_settings,
            dotenv_settings,
            file_secret_settings,
            aws_settings,
        )
```

⚠️ AWS source: nested keys use a `--` separator (e.g. `SqlServer--Password`); arrays aren't supported.

## Azure Key Vault

```python
import os
from azure.identity import DefaultAzureCredential
from pydantic import BaseModel
from pydantic_settings import (
    AzureKeyVaultSettingsSource,
    BaseSettings,
    PydanticBaseSettingsSource,
)

class SubModel(BaseModel):
    a: str

class AzureKeyVaultSettings(BaseSettings):
    foo: str
    bar: int
    sub: SubModel

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls,
        init_settings,
        env_settings,
        dotenv_settings,
        file_secret_settings,
    ):
        az_settings = AzureKeyVaultSettingsSource(
            settings_cls,
            os.environ['AZURE_KEY_VAULT_URL'],
            DefaultAzureCredential(),
        )
        return (
            init_settings,
            env_settings,
            dotenv_settings,
            file_secret_settings,
            az_settings,
        )
```

`AzureKeyVaultSettingsSource(..., snake_case_conversion=True, dash_to_underscore=True)` for naming compatibility.

## pyproject.toml source

```toml
[tool.pydantic]
environment = "production"
database_url = "postgresql://localhost/myapp"
```

```python
from pydantic_settings import BaseSettings, PyprojectTomlSettingsSource

class Settings(BaseSettings):
    environment: str
    database_url: str

    @classmethod
    def settings_customise_sources(
        cls, settings_cls,
        init_settings, env_settings, dotenv_settings, file_secret_settings,
    ):
        return (
            init_settings,
            env_settings,
            PyprojectTomlSettingsSource(settings_cls),
        )
```

## Custom source

```python
from typing import Any
from pydantic.fields import FieldInfo
from pydantic_settings import BaseSettings, PydanticBaseSettingsSource

class CustomSource(PydanticBaseSettingsSource):
    def get_field_value(
        self, field: FieldInfo, field_name: str
    ) -> tuple[Any, str, bool]:
        return None, field_name, False

    def __call__(self) -> dict[str, Any]:
        d: dict[str, Any] = {}
        for field_name, field_info in self.settings_cls.model_fields.items():
            value, _, _ = self.get_field_value(field_info, field_name)
            if value is not None:
                d[field_name] = value
        return d
```

⚠️ **One Settings per service.** Instantiate once (module-level or `@lru_cache`) and inject — re-instantiating per-request hits every source again.

## Cross-references

- [[Pydantic CO 01 - Models]] — `BaseSettings` is a `BaseModel` underneath.
- [[Pydantic CO 02 - Fields]] / [[Pydantic CO 07 - Alias]] — alias mechanics.
- [[Pydantic CO 08 - Configuration]] — `model_config` semantics.
- [[Pydantic CO 09 - Serialization]] — for `model_dump` of settings.
- [[Pydantic CO 10 - Validators]] — `field_validator` for custom decoders.

💡 **Takeaway:** One `Settings(BaseSettings)` class, instantiate once, override `settings_customise_sources` when you need to bend the priority chain.
