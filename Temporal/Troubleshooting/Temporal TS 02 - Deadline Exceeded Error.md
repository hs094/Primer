# TS 02 — Deadline Exceeded Error

🔑 **`Context: deadline exceeded` — a gRPC call from the SDK to the [[Temporal EN 11 - Temporal Service|Temporal Service]] Frontend ran out of time before completing.**

## The error

```
Context: deadline exceeded
```

Often surfaces as:

```
rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

Companion error if clock drift is the cause:

```
Activity complete after timeout
```

## What it means

A gRPC client (your SDK Client or [[Temporal EN 06 - Workers|Worker]]) attached a deadline to a call. The Service did not return in time, so gRPC tore down the call. Source of the deadline: the SDK's per-call timeout, the user-supplied `Context`, or `keepAliveMaxConnectionAge` on the server side.

## Root causes

1. **Network interruptions** — packet loss, NAT timeouts, VPN flaps between Worker → Frontend.
2. **Server overload** — Frontend / History overcommitted; calls queue, then time out.
3. **Rate limiting** — `RpsLimit`, `ConcurrentLimit`, `SystemOverloaded` on the Frontend.
4. **Clock drift** — Worker clock ahead/behind Service clock by enough that timeouts fire prematurely.
5. **Bad config** — wrong target host, wrong `serverName`, expired or mismatched mTLS cert (also see [[Temporal TS 03 - Last Connection Error|TS 03]]).
6. **Long-poll lifetime** — `frontend.keepAliveMaxConnectionAge` shorter than the in-flight request.

## Step 1 — Confirm the Service is reachable

OSS:

```bash
temporal operator cluster health --address 127.0.0.1:7233
```

Cloud: hit the Web UI; if it loads, Frontend RPCs are flowing.

## Step 2 — Check for resource exhaustion

```promql
sum(rate(service_errors_resource_exhausted{}[1m])) by (resource_exhausted_cause)
```

Look for these causes:

- `RpsLimit` — you're past per-namespace RPS quota.
- `ConcurrentLimit` — too many concurrent in-flight RPCs.
- `SystemOverloaded` — Service self-protected; back off.

If any are firing, the fix is upstream of the SDK (raise quotas / scale the cluster), not in your code.

## Step 3 — Sync clocks

Run NTP on every Worker host *and* Service node:

```bash
chronyc tracking      # current offset/drift
timedatectl status    # systemd-based
```

⚠️ A few hundred milliseconds of drift is enough to make Activities look "complete after timeout" or to make deadlines fire before the Service can answer.

## Step 4 — Verify client config

```python
client = await Client.connect(
    "temporal-frontend.prod.internal:7233",
    namespace="prod",
    tls=TLSConfig(
        client_cert=cert_bytes,
        client_private_key=key_bytes,
        server_root_ca_cert=ca_bytes,
        # Domain must match cert SAN exactly
        domain="temporal-frontend.prod.internal",
    ),
)
```

Common slips:
- `domain` / `serverName` doesn't match the cert SAN.
- Wrong port (7233 = Frontend gRPC; not the Web UI port).
- TLS off in client config but Frontend requires it.

## Step 5 — Rule out mTLS mismatches

After enabling mutual TLS:

- Frontend cert SAN matches what clients use as host.
- Internode certs and serverNames are aligned.
- Client CA chain trusts Frontend's cert.

## Step 6 — Tune connection age (if calls always die at the same elapsed time)

If calls reliably fail right around the same duration, look at:

```yaml
frontend:
  keepAliveMaxConnectionAge: 5m   # raise if long-running RPCs need more time
```

⚠️ Bumping this is treating the symptom. If a single RPC needs minutes, the call shape is probably wrong (e.g. a query doing too much work).

## Step 7 — Cloud users

For Temporal Cloud, you cannot access backend logs. File a support ticket with:

- Namespace.
- Workflow ID(s) showing the error.
- Approximate UTC timestamps.
- A `temporal operator cluster health` result for sanity.

## Quick triage flowchart

```
deadline exceeded?
├── reachable?               → no  → network / DNS / firewall / cert
├── resource_exhausted hit?  → yes → RPS/concurrency limits → scale / back off
├── clock drift?             → yes → fix NTP
├── mTLS / config wrong?     → yes → fix domain + cert + CA
└── always fails at same age → tune keepAliveMaxConnectionAge / shorten RPC
```

## Footguns

⚠️ Bumping the SDK timeout to "just retry longer" without checking `resource_exhausted` — you'll just stack more in-flight RPCs and accelerate overload.
⚠️ Treating it as a Workflow problem when it's a network/config problem; the SDK call never reached the Service.
⚠️ Ignoring "Activity complete after timeout" — that's almost always clock drift, not slow Activities.
⚠️ Different mTLS roots between client and Frontend after a CA rotation.

Related: [[Temporal TS 03 - Last Connection Error|TS 03]] (cert expiry), [[Temporal TS 04 - Performance Bottlenecks|TS 04]] (cluster overload), [[Temporal RF 04 - Errors|RF 04]] (error model), [[Temporal SH 08 - Monitoring|SH 08]] (metrics).

💡 **Takeaway:** Deadline-exceeded is almost always Service unreachable, overloaded, clock-drifted, or misconfigured TLS — diagnose those four before touching application code.
