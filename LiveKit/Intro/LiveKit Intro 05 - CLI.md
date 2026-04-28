# 05 — CLI (`lk`)

🔑 `lk` is the terminal entry point for LiveKit — project switching, app scaffolding, agent deploy, doc search.

Source: https://docs.livekit.io/intro/basics/cli/

## Capabilities
- **Project management** — add / list / switch projects, set default for all commands.
- **App templates** — scaffold agents, frontends, token servers with credentials pre-wired.
- **Agent management** — deploy, update, monitor agents on Cloud; secret management, logs, version rollback, status.
- **Docs access** — search and browse LiveKit docs, SDK source, changelogs from the terminal.

## Typical Workflow
```bash
# 1. Auth + project setup
lk cloud auth
lk project add

# 2. Scaffold app from template
lk app create --template voice-assistant

# 3. Deploy agent
lk agent deploy
```

## Notes
- Integrates with LiveKit Cloud for auth + project management.
- Also works against self-hosted servers for local dev.
- Agent ops include secrets, logs, rollback, status checks.

## Tags
[[LiveKit]] [[CLI]] [[Tooling]] [[Agent Deploy]] [[Scaffolding]]
