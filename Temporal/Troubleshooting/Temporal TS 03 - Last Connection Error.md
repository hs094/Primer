# TS 03 — Last Connection Error

🔑 **`Failed reaching server: last connection error` — almost always an expired TLS certificate, occasionally an SDK client racing the Service's role initialization.**

## The error

```
Failed reaching server: last connection error
```

Typical surfacing in client logs / CLI:

```
post "https://...": x509: certificate has expired or is not yet valid
Failed reaching server: last connection error: ...
```

## Root causes

1. **Expired Namespace mTLS certificate** — the dominant cause in Cloud and self-hosted.
2. **Client connects before Service roles initialize** — at cold start, the Frontend may accept connections before History/Matching are ready; very early requests fail.

The docs do *not* list database/persistence outages as a cause for this specific message — those produce different errors ([[Temporal TS 02 - Deadline Exceeded Error|TS 02]] / `Unavailable`).

## Step 1 — Check certificate expiry

For Temporal Cloud Namespaces:

```bash
tcld namespace accepted-client-ca list \
  --namespace <namespace_id>.<account_id> \
  | jq -r '.[0].notAfter'
```

If the returned ISO timestamp is in the past → expired CA. That alone explains the error.

For self-hosted:

```bash
openssl x509 -in client.crt -noout -enddate
openssl x509 -in ca.crt     -noout -enddate
openssl s_client -connect frontend.example.com:7233 -showcerts < /dev/null \
  | openssl x509 -noout -enddate
```

⚠️ The *client* cert, the *CA* cert, and the *Frontend* cert can each expire independently. Check all three.

## Step 2 — Renew

**Cloud-managed CA path:**

1. Generate or obtain a fresh CA / client cert from your existing CA.
2. Self-signed shortcut: `openssl req -x509 -newkey ...` (if your org accepts that for non-prod).
3. Upload via Cloud UI **or**:

```bash
tcld namespace accepted-client-ca add \
  --namespace <ns_id>.<account_id> \
  --ca-certificate-file new-ca.pem
```

4. Roll out the matching client cert + key to every SDK Client and [[Temporal EN 06 - Workers|Worker]].
5. Once all clients are on the new cert, remove the expired CA entry.

**Self-hosted:**

- Reissue from your CA (or self-sign for non-prod).
- Update Frontend / internode TLS config.
- Restart Frontend pods/processes; redeploy Workers with the new client material.

## Step 3 — If certs are valid, suspect a cold-start race

If the cluster (or a single Frontend pod) just started and clients connected immediately:

- Frontend accepts the connection.
- History / Matching aren't fully registered yet → RPC fans out and fails.
- Client reports `last connection error` and gives up.

Mitigations:

- Add a startup probe: SDK client retries with backoff for the first 30–60s.
- In K8s, gate Worker readiness on a successful health check:

```bash
temporal operator cluster health --address temporal-frontend:7233
```

- Make sure Service pods are ready (all roles up) before scaling Workers in.

## Step 4 — Sanity-check what TLS the client is actually presenting

```python
# Python SDK
client = await Client.connect(
    "ns.tmprl.cloud:7233",
    namespace="ns.acct",
    tls=TLSConfig(
        client_cert=Path("client.crt").read_bytes(),
        client_private_key=Path("client.key").read_bytes(),
    ),
)
```

Things to verify:
- The cert file actually corresponds to the key file (`openssl x509 -in client.crt -modulus` vs `openssl rsa -in client.key -modulus` — moduli match).
- The client cert chains up to a CA the Namespace currently accepts.
- The host you're connecting to matches the cert SAN.

## Step 5 — Prevent the next outage

⚠️ Most "last connection error" pages are a renewal-calendar failure, not a software bug.

- Set calendar reminders ~30 days before the *earliest* of: client cert, CA, Frontend cert.
- Alert on cert expiry from Prometheus (e.g. `probe_ssl_earliest_cert_expiry - time() < 30d`).
- Maintain rotation runbook so on-call doesn't have to invent it at 3 AM.
- Use shorter-lived certs *only* if you have automated rotation.

## Triage flowchart

```
"last connection error"
├── tcld / openssl says expired? → renew, deploy new client cert, remove old CA
├── cluster just restarted?      → wait, retry with backoff, check cluster health
├── cert valid + cluster healthy → check host/SAN match, modulus match, CA chain
└── still failing                → file Cloud support ticket / check Frontend logs
```

## Footguns

⚠️ Removing the *old* accepted CA before rolling the new client cert to every Worker — instant outage for anything still on the old cert.
⚠️ Self-signed cert with mismatched CN/SAN → "verified" locally, fails in prod.
⚠️ Client cert and key from different generations → handshake fails with a vague TLS error.
⚠️ Workers with cached TLS material in immutable container images: rotation requires a redeploy, not just a config flag.

Related: [[Temporal TS 02 - Deadline Exceeded Error|TS 02]] (network / overload), [[Temporal PD 01 - Production Overview|PD 01]] (TLS in the prod checklist), [[Temporal SH 08 - Monitoring|SH 08]] (cert expiry alerts).

💡 **Takeaway:** Check cert expiry first, role-init race second. A cert-expiry alert in your monitoring stack prevents 90% of these calls.
