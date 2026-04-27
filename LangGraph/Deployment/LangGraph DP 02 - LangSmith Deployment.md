# DP 02 — LangSmith Deployment

🔑 The Platform CLI builds a Docker image from your `langgraph.json` and either pushes it to LangSmith Deployments or runs it locally with Postgres + Redis sidecars. Same artifact, three runtimes: Cloud, Hybrid, Self-Hosted.

## Deployment modes

| Mode | Who manages | Where it runs |
|---|---|---|
| **Cloud** | LangChain | LangChain infra, GitHub-driven CI/CD |
| **Hybrid** (Enterprise) | LangChain control plane, you host data plane | Your VPC |
| **Self-Hosted** (Enterprise) | You | Your infra end-to-end |

Same runtime, same APIs — only the operator changes.

## Path A: Cloud via GitHub

Push the project (with `langgraph.json` at root) to GitHub → in LangSmith **Deployments → + New Deployment** → connect GitHub, pick repo + branch, submit. Open in [[LangGraph DP 04 - LangSmith Studio|Studio]], grab the deployment URL for SDK calls.

## Path B: Direct deploy from CLI (beta)

```bash
langgraph deploy \
  --name my-agent \
  --deployment-type prod \
  --api-key $LANGSMITH_API_KEY
```

| Flag | Notes |
|---|---|
| `--api-key` | Or env: `LANGGRAPH_HOST_API_KEY` / `LANGSMITH_API_KEY` |
| `--name` | Defaults to current dir name |
| `--deployment-id` | Update an existing deployment |
| `--deployment-type` | `dev` (default) or `prod` for new deployments |
| `--no-wait` | Skip status polling |
| `--verbose` | Stream build logs |

Sub-commands: `langgraph deploy list`, `revisions list <ID>`, `delete <ID>`, `logs [--follow] [--type deploy|build]`.

Prerequisites: Docker running, Buildx for non-x86_64, LangSmith API key with Deployments scope.

## Path C: Self-host with `build` + `up`

### Build the image

```bash
langgraph build -t my-agent:latest
```

Flags: `-t/--tag` (**required**), `--platform linux/amd64,linux/arm64`, `--pull/--no-pull` (default `--pull`), `-c/--config`, `--build-command`/`--install-command` (JS only).

### Run locally with Postgres + Redis

```bash
langgraph up -p 8123 --wait
```

`langgraph up` starts the API container plus the supporting services via an embedded compose file.

| Flag | Default | Notes |
|---|---|---|
| `-p, --port` | `8123` | Exposed port |
| `-c, --config` | `langgraph.json` | |
| `-d, --docker-compose` | — | Extra compose file merged in |
| `--postgres-uri` | bundled local PG | Point at external Postgres |
| `--wait` | — | Block until services healthy |
| `--base-image` | `langchain/langgraph-api` | |
| `--image` | — | Skip build, run pre-built tag |
| `--watch` | — | Restart on file changes |
| `--debugger-port` | — | Attach debugger UI |
| `--pull / --no-pull` | `--pull` | |
| `--recreate / --no-recreate` | `no-recreate` | Force container recreate |

### Generate a Dockerfile

```bash
langgraph dockerfile -c langgraph.json ./Dockerfile
```

For baking the image inside an existing CI/CD pipeline. Override via `dockerfile_lines` in `langgraph.json`.

## Runtime env vars

```bash
LANGSMITH_API_KEY=lsv2_pt_...
LANGSMITH_DEPLOYMENT_NAME=my-agent
LANGGRAPH_HOST_API_KEY=...           # CLI auth
CORS_ALLOW_ORIGINS=https://app.example.com
```

The container needs Postgres (state, threads, runs) and Redis (pubsub, queues). `langgraph up` provisions both; in production, point at managed instances via `--postgres-uri`.

## Locking down routes (`langgraph.json` → `http`)

```json
"http": {
  "disable_meta": false,
  "disable_assistants": false,
  "disable_runs": false,
  "cors": {
    "allow_origins": ["https://app.example.com"],
    "allow_methods": ["GET", "POST"],
    "allow_credentials": true
  }
}
```

`/ok` health-check stays exposed regardless. `api_version: "0.2"` pins the server version (v0.3.7+).

## Storage TTL

```json
"checkpointer": {
  "backend": "default",
  "ttl": {"strategy": "delete", "sweep_interval_minutes": 10, "default_ttl": 43200}
},
"store": {
  "ttl": {"refresh_on_read": true, "default_ttl": 10080, "sweep_interval_minutes": 60}
}
```

## Workflow

[[LangGraph DP 01 - Run a Local Server|`langgraph dev`]] → `langgraph build -t` → `langgraph up --wait` for local Docker parity → `langgraph deploy` (or GitHub push) for the managed plane → verify in [[LangGraph DP 04 - LangSmith Studio|Studio]] and [[LangGraph DP 03 - LangSmith Observability|LangSmith traces]].

💡 Treat `langgraph build` output as the single deployment artifact. Whether it lands on Cloud, your VPC, or a teammate's laptop via `langgraph up`, the runtime contract is identical.
