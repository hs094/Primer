# 05 — Configuration Providers

🔑 Indirect references in any Kafka config file: instead of writing the secret inline, write `${alias:lookup-args}` and a `ConfigProvider` resolves it at runtime. Keeps passwords / API keys out of `broker.properties`, `connect-distributed.properties`, and client configs.

Source: https://kafka.apache.org/42/configuration/configuration-providers/

## Wiring

```properties
# Register one or more providers
config.providers=fileProvider,envVarProvider,dirProvider

config.providers.fileProvider.class=org.apache.kafka.common.config.provider.FileConfigProvider
config.providers.envVarProvider.class=org.apache.kafka.common.config.provider.EnvVarConfigProvider
config.providers.dirProvider.class=org.apache.kafka.common.config.provider.DirectoryConfigProvider

# Optional per-provider params
config.providers.<alias>.param.<name>=<value>
```

## Built-in Providers

| class | placeholder syntax | use case |
| --- | --- | --- |
| `FileConfigProvider` | `${fileProvider:/path/to/file.properties:keyName}` | Single properties file. Read one key at a time. |
| `DirectoryConfigProvider` | `${dirProvider:/path/to/dir:filename}` | Each file in dir = one key (filename), value = file contents. Ideal for K8s `Secret` mounted as a directory. |
| `EnvVarConfigProvider` | `${envVarProvider:VAR_NAME}` | Pull a single env var. |

## Provider-specific params

| key | meaning |
| --- | --- |
| `config.providers.<alias>.param.allowed.paths=/etc/kafka/secrets,/run/secrets` | (File / Directory) Comma-separated allowlist of paths the provider may read. Default = unrestricted. ⚠️ Always set this. |
| `config.providers.<alias>.param.allowlist.pattern=^KAFKA_.*` | (EnvVar) Regex env-var names must match. Default = unrestricted. |

## Example: K8s Secret mounted as directory

K8s mounts a `Secret` at `/etc/kafka/secrets/`, one file per key (`db_password`, `ssl_keystore_password`, ...).

```properties
config.providers=dirProvider
config.providers.dirProvider.class=org.apache.kafka.common.config.provider.DirectoryConfigProvider
config.providers.dirProvider.param.allowed.paths=/etc/kafka/secrets

ssl.keystore.password=${dirProvider:/etc/kafka/secrets:ssl_keystore_password}
ssl.key.password=${dirProvider:/etc/kafka/secrets:ssl_key_password}
```

## Example: Connect connector pulling DB creds from a file

```properties
# In connect-distributed.properties
config.providers=fileProvider
config.providers.fileProvider.class=org.apache.kafka.common.config.provider.FileConfigProvider
config.providers.fileProvider.param.allowed.paths=/etc/connect/credentials
```

```properties
# In a connector POST body
database.user=${fileProvider:/etc/connect/credentials/db.properties:dbUsername}
database.password=${fileProvider:/etc/connect/credentials/db.properties:dbPassword}
```

⚠️ CVE-2024-31141: an attacker who could submit connector configs could exfiltrate arbitrary files via `FileConfigProvider`. In 4.x, lock providers down with `config.providers.<alias>.param.allowed.paths` **and** consider disabling auto-loading in untrusted environments:

```
-Dorg.apache.kafka.automatic.config.providers=none
```

…then explicitly re-enable only what you need.

💡 Custom provider: implement `org.apache.kafka.common.config.provider.ConfigProvider`, package as a JAR, drop on classpath, register the FQCN under `config.providers.<alias>.class`. Useful for Vault, AWS Secrets Manager, GCP Secret Manager.

🧪 Try it: write a single-line `secrets.properties` with `pw=hunter2`, set `password=${fileProvider:/tmp/secrets.properties:pw}` and confirm the broker doesn't log the literal value at INFO.

## Tags
[[Kafka]] [[Brokers]]
