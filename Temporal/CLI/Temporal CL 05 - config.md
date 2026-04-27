# CL 05 — config

🔑 **One-line: read and write CLI **profiles** (address, namespace, TLS paths, API key) stored in `temporal.toml`.**

`temporal config` is the newer, profile-based config system. Each profile is a named bundle of connection settings that you select with `--profile`. Pair with `temporal env` (older YAML system) for legacy setups.

## Subcommands

| Subcommand | Purpose |
|---|---|
| `list` | Show all profile names in the config file |
| `get` | Print one property or the whole active profile |
| `set` | Write a property to a profile |
| `delete` | Remove a single property from a profile |
| `delete-profile` | Remove the entire profile (requires explicit `--profile`) |

## Flags (per subcommand)

| Flag | Where | Purpose |
|---|---|---|
| `--prop, -p` | get/set/delete | Property name (e.g. `address`, `namespace`, `tls.client_cert_path`) |
| `--value, -v` | set | Property value |
| `--profile` | all | Target profile (defaults to active) |

## Examples

```bash
# Show every profile defined
temporal config list

# Inspect the active profile in full
temporal config get

# Single property
temporal config get --prop address

# Point the active profile at Temporal Cloud
temporal config set --prop address \
  --value us-west-2.aws.api.temporal.io:7233
temporal config set --prop namespace --value my-ns.abcde
temporal config set --prop tls.client_cert_path --value ./tls.pem
temporal config set --prop tls.client_key_path  --value ./tls.key

# Drop a single property
temporal config delete --prop tls.client_cert_path

# Remove a whole profile (must specify --profile)
temporal config delete-profile --profile staging
```

## Config file location

Default path:

```
$CONFIG_PATH/temporalio/temporal.toml
```

Where `$CONFIG_PATH` is:

| OS | Path |
|---|---|
| Linux | `$HOME/.config` |
| macOS | `$HOME/Library/Application Support` |
| Windows | `%AppData%` |

It's a plain TOML file — `cat`, edit, and check it into a dotfiles repo (without secret material).

## Common patterns

```bash
# Quick switch between local + cloud
temporal --profile local workflow list
temporal --profile prod  workflow list

# CI: write a profile from secrets, then use it
temporal config set --profile ci --prop address       --value $TEMPORAL_ADDRESS
temporal config set --profile ci --prop namespace     --value $TEMPORAL_NAMESPACE
temporal config set --profile ci --prop api-key       --value $TEMPORAL_API_KEY
temporal --profile ci workflow list
```

⚠️ `delete-profile` **requires** `--profile` to be set explicitly — otherwise it errors. This is intentional: prevents accidental nuke of the default profile.

⚠️ Profile values are stored in plain text. TLS keys and API keys end up on disk readable by your user — protect with filesystem perms, don't sync to public repos.

⚠️ `temporal config` is distinct from `temporal env` ([[Temporal CL 06 - env]]). They write **different files** (`temporal.toml` vs `temporal.yaml`). Pick one and stick with it.

💡 **Takeaway:** one TOML file, many profiles, switch with `--profile`. Use `set/get/delete` for individual props and `delete-profile` to clean up.
