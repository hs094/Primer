# LangGraph Knowledge Pack

Crisp, scannable revision notes covering the LangGraph (Python) docs at docs.langchain.com.
Source: https://docs.langchain.com/oss/python/langgraph/overview

LangGraph is a low-level orchestration framework and runtime for building, managing, and deploying long-running, stateful agents — trusted in production at Klarna, Uber, J.P. Morgan, and others. It is focused entirely on agent orchestration: durable execution, streaming, human-in-the-loop, and persistent memory, without abstracting prompts or architecture.

## How to use
- Each note = one concept, optimized for recall.
- Code blocks are the minimum that exercises the feature.
- 🔑 = core idea. ⚠️ = gotcha. 💡 = mental model.

## Getting Started

| # | Topic | Note |
|---|---|---|
| 01 | What LangGraph is, core benefits, ecosystem | [[Getting Started/LangGraph GS 01 - Overview]] |
| 02 | Install `langgraph`, hello-world graph | [[Getting Started/LangGraph GS 02 - Install LangGraph]] |
| 03 | First real agent end-to-end | [[Getting Started/LangGraph GS 03 - Quickstart]] |
| 04 | Graph API vs Functional API — when to pick which | [[Getting Started/LangGraph GS 04 - Choosing Graph vs Functional APIs]] |
| 05 | Mental model — nodes, edges, state, channels | [[Getting Started/LangGraph GS 05 - Thinking in LangGraph]] |

## Concepts (Core)

| # | Topic | Note |
|---|---|---|
| 01 | Workflows vs agents, router / orchestrator-worker / evaluator-optimizer patterns | [[Concepts/LangGraph CO 01 - Workflows and Agents]] |
| 02 | Application structure, project layout, deployable units | [[Concepts/LangGraph CO 02 - Application Structure]] |
| 03 | Pregel runtime — supersteps, channels, message passing | [[Concepts/LangGraph CO 03 - Pregel Runtime]] |
| 04 | Durable execution, resumability, failure semantics | [[Concepts/LangGraph CO 04 - Durable Execution]] |
| 05 | Persistence with checkpointers (in-memory, SQLite, Postgres) | [[Concepts/LangGraph CO 05 - Persistence (Checkpointers)]] |
| 06 | Short-term + long-term memory, thread vs cross-thread | [[Concepts/LangGraph CO 06 - Memory]] |

## Graph API

| # | Topic | Note |
|---|---|---|
| 01 | `StateGraph`, nodes, edges, conditional edges, `START` / `END` | [[Graph API/LangGraph GA 01 - Graph API Overview]] |
| 02 | Defining state, reducers, `add_messages`, compiling | [[Graph API/LangGraph GA 02 - Using the Graph API]] |
| 03 | Subgraphs — composition, shared vs separate state | [[Graph API/LangGraph GA 03 - Subgraphs]] |

## Functional API

| # | Topic | Note |
|---|---|---|
| 01 | `@entrypoint`, `@task`, when to prefer it over Graph API | [[Functional API/LangGraph FA 01 - Functional API Overview]] |
| 02 | Composing tasks, parallelism, returning state | [[Functional API/LangGraph FA 02 - Using the Functional API]] |

## Patterns

| # | Topic | Note |
|---|---|---|
| 01 | Streaming — `values`, `updates`, `messages`, `custom`, `debug` modes | [[Patterns/LangGraph PT 01 - Streaming]] |
| 02 | Interrupts — human-in-the-loop, `interrupt()`, `Command(resume=...)` | [[Patterns/LangGraph PT 02 - Interrupts (Human-in-the-Loop)]] |
| 03 | Time travel — checkpoint history, fork from past state | [[Patterns/LangGraph PT 03 - Time Travel]] |

## Tutorials

| # | Topic | Note |
|---|---|---|
| 01 | Build a custom RAG agent end-to-end | [[Tutorials/LangGraph TU 01 - Build a Custom RAG Agent]] |

## Deployment

| # | Topic | Note |
|---|---|---|
| 01 | `langgraph dev` — run a local server | [[Deployment/LangGraph DP 01 - Run a Local Server]] |
| 02 | LangSmith Deployment — managed hosting | [[Deployment/LangGraph DP 02 - LangSmith Deployment]] |
| 03 | LangSmith Observability — tracing, evals | [[Deployment/LangGraph DP 03 - LangSmith Observability]] |
| 04 | LangSmith Studio — visual prototyping | [[Deployment/LangGraph DP 04 - LangSmith Studio]] |

## Testing

| # | Topic | Note |
|---|---|---|
| 01 | Testing graphs and agents | [[Testing/LangGraph TS 01 - Testing]] |
| 02 | Agent Chat UI — local debugging frontend | [[Testing/LangGraph TS 02 - Agent Chat UI]] |

## Reference

- [[Reference/LangGraph Case Studies]] — production deployments by industry & use case

## Cross-links

🔑 The five-minute path: [[Getting Started/LangGraph GS 01 - Overview]] → [[Getting Started/LangGraph GS 05 - Thinking in LangGraph]] → [[Graph API/LangGraph GA 01 - Graph API Overview]] → [[Patterns/LangGraph PT 01 - Streaming]].

💡 If something fails at runtime, the answer is almost always in [[Concepts/LangGraph CO 03 - Pregel Runtime]], [[Concepts/LangGraph CO 04 - Durable Execution]], or [[Concepts/LangGraph CO 05 - Persistence (Checkpointers)]] — start there before re-reading the API docs.

⚠️ LangGraph is intentionally low-level. If you want prebuilt agent loops, reach for LangChain's `agents` first; drop down to LangGraph when you need durable execution, interrupts, time travel, or custom orchestration.
