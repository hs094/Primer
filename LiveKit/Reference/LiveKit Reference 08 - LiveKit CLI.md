# 08 — LiveKit CLI

🔑 `lk` is the terminal control plane — manages Cloud projects, scaffolds apps from templates, deploys agents, and exposes raw room/token operations.

Source: https://docs.livekit.io/home/cli/cli-setup/ (canonical /reference/cli/ returns 404)

## Install

```bash
# macOS
brew install livekit-cli

# Linux / WSL
curl -sSL https://get.livekit.io/cli | bash

# Go
go install github.com/livekit/livekit-cli/v2/cmd/lk@latest
```

## Authenticate

```bash
lk cloud auth                    # opens browser, links Cloud project
lk project list
lk project set-default <name>
```

For self-hosted, set env vars or pass flags:

```bash
export LIVEKIT_URL=ws://localhost:7880
export LIVEKIT_API_KEY=devkey
export LIVEKIT_API_SECRET=secret
```

## Project management

```bash
lk project add --name prod --url wss://prod.livekit.cloud --api-key K --api-secret S
lk project list
lk project remove prod
```

## App templates

```bash
lk app create my-agent --template voice-pipeline-agent-python
lk app list-templates
lk app env my-agent          # injects credentials into .env.local
```

## Agent deployment (Cloud)

```bash
lk agent create --silent      # registers from current dir
lk agent deploy
lk agent status
lk agent logs --tail
lk agent versions
lk agent rollback <version>
lk agent secrets set OPENAI_API_KEY=sk-...
```

## Room / participant ops

```bash
lk room list
lk room create my-room
lk room delete my-room
lk room participants list --room my-room
lk room remove-participant --room my-room --identity user-1
```

## Token generation

```bash
lk token create \
  --identity alice \
  --name "Alice" \
  --room my-room \
  --join \
  --valid-for 1h
```

## Docs search

```bash
lk docs search "egress room composite"
```

## Tags
[[LiveKit]] [[Reference]] [[CLI]] [[Tooling]]
