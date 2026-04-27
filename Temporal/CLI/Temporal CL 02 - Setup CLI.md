# CL 02 — Setup CLI

🔑 **One-line: install the `temporal` binary and bootstrap a local dev Temporal Service.**

The CLI is a single Go binary. Install once, then use `temporal server start-dev` to get a local gRPC service + Web UI for SDK development without any Docker stack.

## Installation

| Platform | Method |
|---|---|
| macOS | `brew install temporal` |
| Linux | `brew install temporal` (if available) or `snap install temporal` |
| Linux/macOS/Windows | CDN tarballs at `temporal.download` (amd64/arm64) |
| Docker | `docker run --rm temporalio/temporal --help` |

Verify after install:

```bash
temporal --version
temporal --help
```

## Run the dev server

The local server is a single-process [[Temporal EN 11 - Temporal Service]] with embedded SQLite, Web UI, and a default [[Temporal EN 12 - Namespaces|namespace]]:

```bash
# In-memory (state lost on shutdown)
temporal server start-dev

# Persistent state via SQLite
temporal server start-dev --db-filename ./temporal.db
```

Default endpoints:
- gRPC service: `localhost:7233`
- Web UI: `http://localhost:8233`

To expose the dev server from a Docker container:

```bash
docker run --rm -p 7233:7233 -p 8233:8233 \
  temporalio/temporal server start-dev --ip 0.0.0.0
```

See [[Temporal CL 09 - server]] for all `start-dev` flags (extra namespaces, search attributes, dynamic config, ports).

## CLI help

Help is recursive — every command and subcommand accepts `--help`:

```bash
temporal --help
temporal workflow --help
temporal workflow start --help
temporal workflow update execute --help
```

## Connecting to Temporal Cloud

Once you have client certs, store them as a profile so you don't repeat `--tls-*` flags:

```bash
temporal env set --env prod --key address       --value my-ns.abcde.tmprl.cloud:7233
temporal env set --env prod --key namespace     --value my-ns.abcde
temporal env set --env prod --key tls-cert-path --value ./tls.pem
temporal env set --env prod --key tls-key-path  --value ./tls.key

temporal --env prod workflow list
```

Or use `temporal config` for the newer profile system — see [[Temporal CL 05 - config]] and [[Temporal CL 06 - env]].

## Shell completions

The binary self-installs completions. For bash, zsh, fish, powershell:

```bash
temporal completion zsh > "${fpath[1]}/_temporal"
# or
source <(temporal completion bash)
```

## Common patterns

```bash
# Smoke test a fresh install
temporal server start-dev &
temporal operator cluster health
temporal workflow list  # empty list is success

# Pre-register search attributes for an SDK sample
temporal server start-dev \
  --search-attribute CustomKeywordField=Keyword \
  --search-attribute CustomIntField=Int
```

⚠️ `temporal server start-dev` without `--db-filename` keeps everything in memory. Restart = wiped Workflow history. Always pass `--db-filename` when iterating on long-running flows.

⚠️ The dev server is **explicitly not for production**. For self-hosted production deployments see [[Temporal SH 01 - Self-Hosted Overview]].

💡 **Takeaway:** install with brew/snap/curl, run `temporal server start-dev --db-filename`, point your SDK at `localhost:7233`, and you're coding.
