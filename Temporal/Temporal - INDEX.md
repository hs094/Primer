# Temporal Knowledge Pack

Refresher vault mirroring [docs.temporal.io](https://docs.temporal.io). Code-first, terse — for re-reading, not first-time learning.

🔑 **Why Temporal:** durable execution as a primitive — Workflows survive crashes, retries, and waits of arbitrary length, with `@workflow.defn` / `@activity.defn` as the entire programming model. Event History is ground truth, the Replayer is the time machine, and the Service handles the rest.

## Get Started

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal GS 01 - Welcome and Why]] | What Temporal is and the case for using it |
| 02 | [[Temporal GS 02 - Understanding Temporal]] | Mental model — durable execution, Event History, replay |
| 03 | [[Temporal GS 03 - Features]] | Feature catalog: schedules, signals, queries, retries, etc. |
| 04 | [[Temporal GS 04 - Use Cases and Design Patterns]] | When to reach for Temporal; common patterns |
| 05 | [[Temporal GS 05 - Set Up Local Python]] | `pip install temporalio`, Hello World, dev server |

## Encyclopedia (concepts)

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal EN 01 - Temporal Platform]] | Platform overview — Service + SDKs + UI |
| 02 | [[Temporal EN 02 - Temporal SDKs]] | What an SDK does; per-language overview |
| 03 | [[Temporal EN 03 - Workflows]] | Workflow Definition / Execution / determinism |
| 04 | [[Temporal EN 04 - Activities]] | Activity Definition / Execution / Tasks |
| 05 | [[Temporal EN 05 - Detecting Application Failures]] | Retries, `ApplicationError`, non-retryable |
| 06 | [[Temporal EN 06 - Workers]] | Worker process, Task polling, slot model |
| 07 | [[Temporal EN 07 - Event History]] | The append-only log that replay walks |
| 08 | [[Temporal EN 08 - Workflow Message Passing]] | Signals, Queries, Updates — concept layer |
| 09 | [[Temporal EN 09 - Child Workflows]] | When to spawn a Child vs. start a top-level |
| 10 | [[Temporal EN 10 - Visibility]] | List Filters, Search Attributes, the visibility store |
| 11 | [[Temporal EN 11 - Temporal Service]] | Frontend, History, Matching, Worker; persistence |
| 12 | [[Temporal EN 12 - Namespaces]] | Isolation unit, retention, defaults |
| 13 | [[Temporal EN 13 - Temporal Nexus]] | Cross-namespace typed RPC |
| 14 | [[Temporal EN 14 - Extensibility]] | Data Conversion, Codecs, Interceptors |

## Develop (Python SDK)

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal DV 01 - Python SDK Overview]] | The full Develop guide entry point |
| 02 | [[Temporal DV 02 - Workflow Basics]] | `@workflow.defn`, `@workflow.run`, structure |
| 03 | [[Temporal DV 03 - Child Workflows]] | `start_child_workflow` / `execute_child_workflow` |
| 04 | [[Temporal DV 04 - Continue As New]] | `continue_as_new` for unbounded loops |
| 05 | [[Temporal DV 05 - Workflow Cancellation]] | Cancellation propagation, `asyncio.shield` |
| 06 | [[Temporal DV 06 - Workflow Timeouts]] | execution / run / task timeouts |
| 07 | [[Temporal DV 07 - Message Passing]] | Signals, Queries, Updates in Python |
| 08 | [[Temporal DV 08 - Schedules]] | `Client.create_schedule`, intervals, backfill |
| 09 | [[Temporal DV 09 - Timers]] | `asyncio.sleep` as a durable Timer |
| 10 | [[Temporal DV 10 - Activity Basics]] | `@activity.defn`, sync vs async, types |
| 11 | [[Temporal DV 11 - Activity Execution]] | `execute_activity` / `start_activity` from Workflows |
| 12 | [[Temporal DV 12 - Standalone Activities]] | Activities run directly from a Client |
| 13 | [[Temporal DV 13 - Activity Timeouts]] | start-to-close, schedule-to-*, heartbeat |
| 14 | [[Temporal DV 14 - Async Activity Completion]] | Complete an Activity from outside the Worker |
| 15 | [[Temporal DV 15 - Worker Processes]] | `Worker(client, task_queue, workflows, activities)` |
| 16 | [[Temporal DV 16 - Temporal Client]] | `Client.connect`, start/execute, handles |
| 17 | [[Temporal DV 17 - Nexus]] | Service contracts, sync/async ops, cross-ns calls |
| 18 | [[Temporal DV 18 - Observability]] | Metrics, OTel traces, logs, queries |
| 19 | [[Temporal DV 19 - Testing]] | `WorkflowEnvironment`, time-skip, Replayer |
| 20 | [[Temporal DV 20 - Sandbox and Sync vs Async]] | Sandbox restrictions, sync activity threading |
| 21 | [[Temporal DV 21 - Data Handling]] | Data converter, codecs, Pydantic interop |
| 22 | [[Temporal DV 22 - Debugging]] | UI / `temporal` CLI / Replayer / stack-trace query |

## CLI (`temporal`)

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal CL 01 - CLI Overview]] | Overall command map and global flags |
| 02 | [[Temporal CL 02 - Setup CLI]] | Install, dev server (`temporal server start-dev`), env |
| 03 | [[Temporal CL 03 - activity]] | `complete`, `fail` — ad-hoc Activity finalization |
| 04 | [[Temporal CL 04 - batch]] | `list`, `describe`, `terminate` for batch jobs |
| 05 | [[Temporal CL 05 - config]] | `temporal` CLI configuration profiles |
| 06 | [[Temporal CL 06 - env]] | Named environments and the active one |
| 07 | [[Temporal CL 07 - operator]] | `namespace`, `cluster`, `nexus`, `search-attribute` |
| 08 | [[Temporal CL 08 - schedule]] | `create`, `describe`, `trigger`, `backfill`, `delete` |
| 09 | [[Temporal CL 09 - server]] | `start-dev` (single binary, no deps) |
| 10 | [[Temporal CL 10 - task-queue]] | `describe`, `list-partition`, versioning ops |
| 11 | [[Temporal CL 11 - worker]] | Worker introspection commands |
| 12 | [[Temporal CL 12 - workflow]] | `start`, `execute`, `list`, `signal`, `query`, `show`, `stack`, `terminate`, `reset` |

## Cloud

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal TC 01 - Cloud Overview]] | What Temporal Cloud is and what you skip when using it |
| 02 | [[Temporal TC 02 - Get Started Cloud]] | Sign-up flow, namespace creation, first connection |
| 03 | [[Temporal TC 03 - Cloud Namespaces]] | Cloud-specific namespace model and limits |
| 04 | [[Temporal TC 04 - API Keys]] | Bearer auth — create, rotate, scope |
| 05 | [[Temporal TC 05 - mTLS Certificates]] | Cert-based auth, CA upload, renewal |
| 06 | [[Temporal TC 06 - Account Access]] | Account-level access model |
| 07 | [[Temporal TC 07 - Roles and Permissions]] | Role catalog and what each can do |
| 08 | [[Temporal TC 08 - Users and Groups]] | Users, user groups, service accounts |
| 09 | [[Temporal TC 09 - SAML and SCIM]] | SSO and provisioning |
| 10 | [[Temporal TC 10 - Cloud Metrics]] | OpenMetrics endpoint, `temporal_cloud_v1_*` |
| 11 | [[Temporal TC 11 - Billing and Usage]] | Actions, storage, Billing API, Usage Dashboards |
| 12 | [[Temporal TC 12 - Connectivity]] | PrivateLink, IP allowlists, regions |
| 13 | [[Temporal TC 13 - High Availability]] | Multi-region replication, RPO/RTO |
| 14 | [[Temporal TC 14 - Cloud Operations and tcld]] | Cloud Ops API, `tcld`, Terraform, audit, notifications, export |

## Production

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal PD 01 - Production Overview]] | What you have to think about for prod |
| 02 | [[Temporal PD 02 - Worker Deployments]] | Worker Deployments, versioning, ramping |
| 03 | [[Temporal PD 03 - Codecs and Encryption]] | Payload codecs, the codec server, key rotation |

## Self-Hosted

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal SH 01 - Self-Hosted Overview]] | Running your own Temporal Service |
| 02 | [[Temporal SH 02 - Deployment]] | Topology — Frontend / History / Matching / Worker |
| 03 | [[Temporal SH 03 - Embedded Server]] | Embedding the Service inside your binary |
| 04 | [[Temporal SH 04 - Configurable Defaults]] | Defaults and how to override |
| 05 | [[Temporal SH 05 - Production Checklist]] | What to check before you flip to prod |
| 06 | [[Temporal SH 06 - Self-Hosted Namespaces]] | Creating, configuring, retention |
| 07 | [[Temporal SH 07 - Security]] | mTLS, authz, encryption at rest |
| 08 | [[Temporal SH 08 - Monitoring]] | Cluster metrics, dashboards, alerts |
| 09 | [[Temporal SH 09 - Self-Hosted Visibility]] | Standard vs advanced visibility, ES/OS |
| 10 | [[Temporal SH 10 - Upgrade Server]] | Rolling upgrades, schema migrations |
| 11 | [[Temporal SH 11 - Archival and Replication]] | History archival, multi-cluster replication |

## Best Practices

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal BP 01 - Best Practices Overview]] | The catalog and how to use it |
| 02 | [[Temporal BP 02 - Worker Performance]] | Slot tuning, sticky cache, Activity throughput |
| 03 | [[Temporal BP 03 - Pre-Production Testing]] | Replay-against-prod-histories, staging |
| 04 | [[Temporal BP 04 - Multi-Tenant Patterns]] | Namespaces vs Search Attributes for tenancy |
| 05 | [[Temporal BP 05 - Managing Namespaces]] | When to split, retention defaults |
| 06 | [[Temporal BP 06 - Managing APS Limits]] | Actions-per-second budgeting |
| 07 | [[Temporal BP 07 - Cloud Access Control]] | Roles, service accounts, least privilege |
| 08 | [[Temporal BP 08 - Security Controls]] | Cloud security toggles and recommendations |
| 09 | [[Temporal BP 09 - Cost Optimization]] | Knobs that drive the bill |

## Troubleshooting

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal TS 01 - Blob Size Limit Error]] | Payload too large; how to split |
| 02 | [[Temporal TS 02 - Deadline Exceeded Error]] | gRPC deadline; root causes |
| 03 | [[Temporal TS 03 - Last Connection Error]] | Service unreachable from Worker / Client |
| 04 | [[Temporal TS 04 - Performance Bottlenecks]] | Schedule-to-start, replay, sticky cache |

## References

| # | Note | What it covers |
|---|---|---|
| 01 | [[Temporal RF 01 - SDK Metrics]] | The full SDK metric catalog |
| 02 | [[Temporal RF 02 - Commands]] | Workflow Command catalog |
| 03 | [[Temporal RF 03 - Events]] | Event types in History |
| 04 | [[Temporal RF 04 - Errors]] | Workflow Task error reasons |
| 05 | [[Temporal RF 05 - Failures]] | Failure type taxonomy |
| 06 | [[Temporal RF 06 - Service Configuration]] | Static service config keys |
| 07 | [[Temporal RF 07 - Dynamic Configuration]] | Runtime-tunable keys |

💡 **Takeaway:** start at [[Temporal EN 03 - Workflows]] / [[Temporal EN 04 - Activities]] / [[Temporal EN 06 - Workers]] for the mental model, [[Temporal DV 02 - Workflow Basics]] / [[Temporal DV 10 - Activity Basics]] / [[Temporal DV 15 - Worker Processes]] for Python code, [[Temporal CL 12 - workflow]] for the CLI surface, [[Temporal DV 22 - Debugging]] when something's stuck, and [[Temporal BP 02 - Worker Performance]] when something's slow.
