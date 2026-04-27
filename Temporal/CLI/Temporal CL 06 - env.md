# CL 06 — env

🔑 **One-line: read and write **environment presets** (named bundles of CLI flags) stored in `temporal.yaml`.**

`temporal env` is the original CLI config system — predates `temporal config`. Each environment is a YAML map of `key: value` pairs that, when keys match CLI flag names, auto-populate flags. Select with `--env`.

## Subcommands

| Subcommand | Purpose |
|---|---|
| `list` | Show all environments configured locally |
| `get` | Print all keys in an environment, or one key |
| `set` | Add/update a key in an environment |
| `delete` | Remove an environment, or one key from it |

## Flags

| Flag | Where | Purpose |
|---|---|---|
| `--env` | all | Environment name (defaults to `default` if omitted) |
| `--key, -k` | get/set/delete | Property name |
| `--value, -v` | set | Property value |

## Examples

```bash
# List every env on this box
temporal env list

# Dump the prod env
temporal env get --env prod

# One key
temporal env get --env prod --key namespace

# Configure prod once
temporal env set --env prod --key address       --value my-ns.tmprl.cloud:7233
temporal env set --env prod --key namespace     --value my-ns.abcde
temporal env set --env prod --key tls-cert-path --value ./tls.pem
temporal env set --env prod --key tls-key-path  --value ./tls.key

# Use it
temporal --env prod workflow list

# Drop one key
temporal env delete --env prod --key tls-key-path

# Drop the whole env
temporal env delete --env staging
```

## Env file location

```
$HOME/.config/temporalio/temporal.yaml
```

Override with `--env-file <path>`. The YAML structure is one top-level map per env name; keys mirror CLI flag long names (with `-` instead of `--`):

```yaml
default:
  address: localhost:7233
  namespace: default
prod:
  address: my-ns.tmprl.cloud:7233
  namespace: my-ns.abcde
  tls-cert-path: /Users/me/.tmprl/tls.pem
  tls-key-path:  /Users/me/.tmprl/tls.key
```

## Common patterns

```bash
# Create a "cloud" env distinct from local
temporal env set --env cloud --key address    --value my-ns.tmprl.cloud:7233
temporal env set --env cloud --key namespace  --value my-ns.abcde
temporal env set --env cloud --key api-key    --value $TEMPORAL_API_KEY

# Daily flow
temporal --env cloud workflow list
temporal --env cloud schedule list
```

⚠️ Storing keys with names that **don't** match a real CLI flag is silently allowed — they're just dead values. Typo `namspace` and you'll wonder why `--namespace` isn't being picked up.

⚠️ `delete` without `--key` removes the **entire** environment (targets `default` if `--env` is omitted). Always pass `--env` explicitly when deleting.

⚠️ `temporal env` and `temporal config` ([[Temporal CL 05 - config]]) are independent. Mixing both leads to confusion about which file actually populates a given flag — pick one as your source of truth.

💡 **Takeaway:** `temporal env set --env <name> --key <flag> --value <val>` once, then `temporal --env <name> ...` everywhere.
