# PD 02 — Worker Deployments

🔑 **Worker Versioning pins each Workflow to a Worker Deployment Version so you can roll out new Workflow code without breaking in-flight executions.**

## The problem it solves

Workflow code must be deterministic across replays. If you redeploy a Worker with new branches, an old Workflow that was already partway through can replay against new code and hit a non-determinism error. Historical mitigations:

- `patched(...)` / `GetVersion(...)` — works, but litters the codebase forever.
- New Task Queue per release — possible, painful to operate.

**Worker Versioning** replaces both as the recommended default. Each Worker reports a **Worker Deployment Version** to the Service; Workflows are pinned to a version and only that version's Workers replay them.

## Core concepts

| Term | What it is |
|---|---|
| Worker Deployment | A logical group of Workers serving a Task Queue (e.g. `payments`). |
| Worker Deployment Version | An immutable build identity within that Deployment (e.g. `payments.v42`). |
| Current Version | The version that picks up *new* Workflow Executions. |
| Ramping Version | A version receiving a percentage of new starts (canary). |
| Pinned | A Workflow stays on the version that started it for its entire life. |
| AutoUpgrade | A Workflow can move to the new Current Version on next Workflow Task. |
| Drainage | Old version still alive long enough to finish its in-flight Workflows. |

## Lifecycle of a release

```
  v42 (Current)            v42 (Current)        v43 (Current)
  ─────────────  ramp 10%  ──────────────       ──────────────
                 v43       v42 (Current)        v42 (Draining)
                           v43 (Ramping 50%)    v43 picks up new
```

1. Deploy `v43` Workers alongside `v42`. They register but get **no new starts**.
2. Set `v43` as **Ramping** at e.g. 10% — 10% of new Workflow starts go to `v43`.
3. Increase ramp progressively (10 → 25 → 50 → 100).
4. Set `v43` as **Current**. `v42` enters **Draining**.
5. When `v42` has zero open Pinned Workflows (or after retention), retire its Workers.

## Pinned vs AutoUpgrade

- **Pinned (default for new Workflows under versioning)** — full safety. A Workflow started on `v42` *only* ever runs on `v42` Workers. Old code stays alive until that Workflow closes.
- **AutoUpgrade** — opt-in per Workflow type when you know the change is replay-safe. Workflow can hop to the new Current Version on its next Workflow Task.

⚠️ AutoUpgrade is *not* a free pass. The new code still must be deterministic-compatible with the existing Event History.

## Ramping mechanics

- Ramp percent applies to **new** Workflow Executions, not in-flight ones.
- Signals/queries continue to land on whichever version owns the Workflow.
- Ramps are set at the Deployment level via CLI / SDK / UI.

```bash
temporal worker-deployment set-ramping-version \
  --deployment-name payments \
  --version v43 \
  --percentage 10
```

```bash
temporal worker-deployment set-current-version \
  --deployment-name payments \
  --version v43
```

## Drainage and retention

When a version stops being Current and stops Ramping, it enters **Draining**:

- Service tracks open Pinned Workflows on it.
- Workers on the draining version keep polling until those Workflows complete.
- After drainage + Namespace retention period, the version is safe to delete.

⚠️ Do not scale draining-version Workers to zero. Pinned Workflows on that version will sit unable to make progress and eventually time out.

## Deploying the Workers themselves

You still ship Worker processes the usual way:

- **Containers** — one image per version, tag with the version string.
- **Kubernetes** — separate Deployment/ReplicaSet per version; the **Worker Controller** can manage scale and lifecycle programmatically.
- **EKS path** — build image, push to ECR, deploy as a K8s Deployment with a Service-side rollout.

```python
worker = Worker(
    client,
    task_queue="payments",
    workflows=[PaymentWorkflow],
    activities=[charge_card],
    deployment_options=WorkerDeploymentOptions(
        deployment_name="payments",
        build_id="v43",
        use_worker_versioning=True,
    ),
)
```

(Names of options vary by SDK; the trio is always *deployment name + build/version id + opt-in flag*.)

## Without versioning (the fallback)

If you cannot adopt versioning yet:

- Use `patched(...)` / `GetVersion(...)` to branch on event-history compatibility.
- Or run a brand-new Task Queue per breaking release and migrate clients.

⚠️ Both fallbacks bake migration debt into your codebase. Worker Versioning replaces them; new deployments should prefer it.

## Operational checklist per release

- [ ] New version registered (visible in Web UI under Worker Deployments).
- [ ] Workers for the new version healthy, polling, slots not saturated ([[Temporal BP 02 - Worker Performance|BP 02]]).
- [ ] Ramp at low % first; watch [[Temporal RF 04 - Errors|errors]] / [[Temporal RF 05 - Failures|failures]] dashboards.
- [ ] Promote to Current.
- [ ] Old version drains cleanly (zero open Pinned Workflows).
- [ ] Retire old Workers only after drainage *and* retention window.

## Common footguns

⚠️ Killing an old version before it drains → orphaned Pinned Workflows.
⚠️ Setting AutoUpgrade by default for code that's not replay-safe.
⚠️ Forgetting that ramp affects *new* starts only — existing executions don't migrate.
⚠️ Mixing versioned and unversioned Workers on the same Task Queue during cutover; finish one regime before starting the other.

💡 **Takeaway:** Build per-version Worker images, register them as Worker Deployment Versions, ramp new starts gradually, promote to Current, then let the old version drain before you scale it down.
