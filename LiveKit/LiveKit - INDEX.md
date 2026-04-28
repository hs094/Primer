# LiveKit Knowledge Pack

Crisp, scannable revision notes covering the LiveKit docs (docs.livekit.io).
Source: https://docs.livekit.io/

## How to use
- Each note = one concept, optimized for recall.
- Code blocks are the minimum that exercises the feature.
- 🔑 = core idea on every note's first line.
- Some Source URLs were rewritten where the canonical path 404'd at fetch time; the actual fetched URL is on each note.

## Top-level

| Topic | Note |
|---|---|
| What LiveKit is — server, SDKs, agents, cloud | [[LiveKit 01 - Overview]] |

## Intro (`/intro/`)

| # | Topic | Note |
|---|---|---|
| 01 | Project, mission, license, ecosystem | [[Intro/LiveKit Intro 01 - About]] |
| 02 | Core concepts — Room, Participant, Track | [[Intro/LiveKit Intro 02 - Basics]] |
| 03 | LiveKit Cloud — hosted SFU, telephony, ops | [[Intro/LiveKit Intro 03 - LiveKit Cloud]] |
| 04 | Connecting clients with tokens + URL | [[Intro/LiveKit Intro 04 - Connecting]] |
| 05 | `livekit-cli` — token gen, room ops, load test | [[Intro/LiveKit Intro 05 - CLI]] |
| 06 | Building AI agents — mental model | [[Intro/LiveKit Intro 06 - Building AI Agents]] |

## Agents — Get Started (`/agents/`)

| # | Topic | Note |
|---|---|---|
| 01 | Agents framework introduction | [[Agents/LiveKit Agents 01 - Introduction]] |
| 02 | Voice AI quickstart (Python / Node) | [[Agents/LiveKit Agents 02 - Voice AI Quickstart]] |
| 03 | Agent Builder — browser playground | [[Agents/LiveKit Agents 03 - Agent Builder]] |
| 04 | Prompting guide for agents | [[Agents/LiveKit Agents 04 - Prompting Guide]] |

## Agents — Multimodality

| # | Topic | Note |
|---|---|---|
| 05 | Speech, text, vision overview | [[Agents/LiveKit Agents 05 - Multimodality Overview]] |
| 06 | Speech & audio I/O, `session.say()` | [[Agents/LiveKit Agents 06 - Speech and Audio]] |
| 07 | Text & transcriptions | [[Agents/LiveKit Agents 07 - Text and Transcriptions]] |
| 08 | Vision — images & video frames | [[Agents/LiveKit Agents 08 - Vision (Images and Video)]] |

## Agents — Logic & Structure

| # | Topic | Note |
|---|---|---|
| 09 | Logic and structure overview | [[Agents/LiveKit Agents 09 - Logic and Structure Overview]] |
| 10 | `AgentSession` lifecycle | [[Agents/LiveKit Agents 10 - Sessions]] |
| 11 | `ChatContext` & message history | [[Agents/LiveKit Agents 11 - Chat Context]] |
| 12 | `function_tool`, `RunContext`, tasks | [[Agents/LiveKit Agents 12 - Tasks and Function Tools]] |
| 13 | Workflows — multi-step orchestration | [[Agents/LiveKit Agents 13 - Workflows]] |
| 14 | Pipeline nodes & lifecycle hooks | [[Agents/LiveKit Agents 14 - Pipeline Nodes and Hooks]] |
| 15 | Turn detection & interruptions | [[Agents/LiveKit Agents 15 - Turn Detection and Interruptions]] |
| 16 | Multi-agent handoffs | [[Agents/LiveKit Agents 16 - Handoffs]] |
| 17 | External data & RAG | [[Agents/LiveKit Agents 17 - External Data and RAG]] |
| 18 | Testing agent behavior | [[Agents/LiveKit Agents 18 - Testing]] |

## Agents — Server & Models

| # | Topic | Note |
|---|---|---|
| 19 | Worker / agent server overview | [[Agents/LiveKit Agents 19 - Agent Server Overview]] |
| 20 | `WorkerOptions`, startup modes | [[Agents/LiveKit Agents 20 - Worker Options]] |
| 21 | `JobContext`, job lifecycle | [[Agents/LiveKit Agents 21 - Job Lifecycle]] |
| 22 | Agent dispatch — room → worker | [[Agents/LiveKit Agents 22 - Agent Dispatch]] |
| 23 | Models — provider matrix | [[Agents/LiveKit Agents 23 - Models Overview]] |
| 24 | LLM integrations | [[Agents/LiveKit Agents 24 - LLM Integrations]] |
| 25 | STT integrations | [[Agents/LiveKit Agents 25 - STT Integrations]] |
| 26 | TTS integrations | [[Agents/LiveKit Agents 26 - TTS Integrations]] |
| 27 | Realtime (speech-to-speech) models | [[Agents/LiveKit Agents 27 - Realtime Models]] |
| 28 | Virtual avatars | [[Agents/LiveKit Agents 28 - Virtual Avatars]] |

## Frontends (`/frontends/`)

| # | Topic | Note |
|---|---|---|
| 01 | Frontend SDKs & component libs | [[Frontends/LiveKit Frontends 01 - Overview]] |
| 02 | React components & hooks | [[Frontends/LiveKit Frontends 02 - React Components]] |
| 03 | iOS / Swift components | [[Frontends/LiveKit Frontends 03 - iOS Swift]] |
| 04 | Android Compose components | [[Frontends/LiveKit Frontends 04 - Android Compose]] |

## Telephony (`/sip/`)

| # | Topic | Note |
|---|---|---|
| 01 | SIP overview, when to use | [[Telephony/LiveKit Telephony 01 - Overview]] |
| 02 | Quickstarts — first call in/out | [[Telephony/LiveKit Telephony 02 - Quickstarts]] |
| 03 | SIP trunks (inbound + outbound) | [[Telephony/LiveKit Telephony 03 - SIP Trunks]] |
| 04 | Inbound calls — accepting & routing | [[Telephony/LiveKit Telephony 04 - Inbound Calls]] |
| 05 | Outbound calls — placing PSTN calls | [[Telephony/LiveKit Telephony 05 - Outbound Calls]] |
| 06 | Dispatch rules — number → room | [[Telephony/LiveKit Telephony 06 - Dispatch Rules]] |
| 07 | DTMF & IVR menus | [[Telephony/LiveKit Telephony 07 - DTMF and IVR]] |
| 08 | Call transfer (cold) | [[Telephony/LiveKit Telephony 08 - Call Transfer]] |

## Transport / WebRTC

| # | Topic | Note |
|---|---|---|
| 01 | WebRTC + SFU overview | [[Transport/LiveKit Transport 01 - WebRTC Overview]] |
| 02 | Connecting clients to a room | [[Transport/LiveKit Transport 02 - Connecting to LiveKit]] |
| 03 | `Room` — events, state, lifecycle | [[Transport/LiveKit Transport 03 - Rooms]] |
| 04 | Participants — local vs remote | [[Transport/LiveKit Transport 04 - Participants]] |
| 05 | Tracks — audio / video / data | [[Transport/LiveKit Transport 05 - Tracks]] |
| 06 | Publishing media (camera, mic, screen) | [[Transport/LiveKit Transport 06 - Publishing Media]] |
| 07 | Subscribing to remote tracks | [[Transport/LiveKit Transport 07 - Subscribing to Media]] |
| 08 | Data messages, streamText, RPC | [[Transport/LiveKit Transport 08 - Data Messages]] |
| 09 | Auth — `AccessToken`, `VideoGrants` | [[Transport/LiveKit Transport 09 - Authentication and Tokens]] |

## Manage & Deploy

| # | Topic | Note |
|---|---|---|
| 01 | Deploying agents — Cloud & K8s | [[Deploy/LiveKit Deploy 01 - Deploying Agents]] |
| 02 | Production best practices | [[Deploy/LiveKit Deploy 02 - Production Best Practices]] |
| 03 | Observability — logs, metrics, traces | [[Deploy/LiveKit Deploy 03 - Observability]] |
| 04 | Self-hosting LiveKit Server (local) | [[Deploy/LiveKit Deploy 04 - Self-Hosting LiveKit Server]] |
| 05 | Distributed deployment | [[Deploy/LiveKit Deploy 05 - Distributed Deployment]] |

## Reference

| # | Topic | Note |
|---|---|---|
| 01 | Recipes — task-shaped examples | [[Reference/LiveKit Reference 01 - Recipes Index]] |
| 02 | Server APIs overview | [[Reference/LiveKit Reference 02 - Server APIs Overview]] |
| 03 | Room Service API | [[Reference/LiveKit Reference 03 - Room Service API]] |
| 04 | Participant management API | [[Reference/LiveKit Reference 04 - Participant Management]] |
| 05 | Egress — recording & live streaming | [[Reference/LiveKit Reference 05 - Egress (Recording and Streaming)]] |
| 06 | Ingress — RTMP / WHIP / URL pulls | [[Reference/LiveKit Reference 06 - Ingress (External Sources)]] |
| 07 | Webhooks — room/participant events | [[Reference/LiveKit Reference 07 - Webhooks]] |
| 08 | LiveKit CLI reference | [[Reference/LiveKit Reference 08 - LiveKit CLI]] |
| 09 | Token generation patterns | [[Reference/LiveKit Reference 09 - Token Generation]] |

## Tags

[[LiveKit]] [[WebRTC]] [[real-time]] [[voice-AI]] [[agents]] [[SIP]] [[Telephony]] [[media-server]]
