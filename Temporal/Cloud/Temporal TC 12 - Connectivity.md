# TC 12 — Connectivity

🔑 **Cloud namespaces speak gRPC over TLS on `:7233`; auth is mTLS or API key; you can keep traffic off the public internet with PrivateLink or Private Service Connect.**

## Connection paths
By default a namespace is reachable over the public internet at a stable gRPC endpoint. Cloud also offers private connectivity:

- **AWS PrivateLink** — VPC endpoint into AWS-hosted namespaces.
- **GCP Private Service Connect** — equivalent for GCP-hosted namespaces.

Whether the path is public or private, **authentication is always mTLS or API keys** — there's no unauthenticated path.

## Endpoint shape
A Cloud Worker or client needs:

- **Host:Port** — the gRPC endpoint with port `7233` (e.g., `my-namespace.acctId.tmprl.cloud:7233`).
- **Namespace** — the form `my-namespace.acct-id`.
- **TLS** — server name override (e.g., `my-namespace.my-account.tmprl.cloud`) plus client cert (mTLS) or `Authorization: Bearer <api-key>` (API key).

### Python (mTLS)
```python
from temporalio.client import Client, TLSConfig

client = await Client.connect(
    "my-namespace.acct123.tmprl.cloud:7233",
    namespace="my-namespace.acct123",
    tls=TLSConfig(
        client_cert=client_cert_pem,
        client_private_key=client_key_pem,
    ),
)
```

### Python (API key)
```python
from temporalio.client import Client

client = await Client.connect(
    "my-namespace.acct123.tmprl.cloud:7233",
    namespace="my-namespace.acct123",
    api_key=os.environ["TEMPORAL_API_KEY"],
    tls=True,
)
```

### Go (PrivateLink endpoint)
```go
c, err := client.Dial(client.Options{
    HostPort: "vpce-0123456789abcdef-abc.us-east-1.vpce.amazonaws.com:7233",
    Namespace: "namespace-name.accId",
    ConnectionOptions: client.ConnectionOptions{
        TLS: &tls.Config{
            Certificates: []tls.Certificate{cert},
            ServerName: "my-namespace.my-account.tmprl.cloud",
        },
    },
})
```

The `ServerName` override is required when dialing a PrivateLink hostname so TLS still validates against the Temporal cert.

## Connectivity rules
**Connectivity rules** are Cloud's mechanism for limiting which network paths can reach a namespace. Each namespace can carry one or more rules:

- **Public** — internet access (the default).
- **Private AWS** — requires a VPC endpoint ID and AWS region.
- **Private GCP** — requires a connection ID, region, and GCP project ID.

Manage rules in the UI, [[Temporal TC 14 - Cloud Operations and tcld]] (`tcld connectivity-rule ...`), or the Cloud Ops API. Removing the public rule turns the namespace into private-only.

## Authentication options
- **mTLS** — long-standing default; client cert/key pair issued from a CA you upload to the namespace. Rotate per your cert policy.
- **API keys** — bearer tokens tied to a User or Service Account, easier to rotate, friendlier for serverless. See [[Temporal TC 08 - Users and Groups]].

Pick one per Worker fleet; mixing isn't required. API keys are the lower-friction default for new deployments.

⚠️ mTLS namespace certs are unrelated to the deprecated mTLS metrics endpoint (see [[Temporal TC 10 - Cloud Metrics]]). Keep their rotation calendars separate so you don't conflate failures.

## Control-plane endpoints
The data plane is the namespace endpoint above. The **control plane** (used by tcld, Terraform, Cloud Ops API, web UI) lives at:

- `saas-api.tmprl.cloud:443` — gRPC + HTTP for Cloud Ops API.
- `web.saas-api.tmprl.cloud` — Web UI traffic.

Both surfaces support public access; AWS PrivateLink is also available for `saas-api.tmprl.cloud` so your operations tooling stays inside your VPC.

## Troubleshooting checklist
- TLS handshake failure → server name override missing or wrong CA.
- `Unauthenticated` → API key wrong or expired; cert revoked or expired.
- `PermissionDenied` → identity exists but lacks namespace permission; see [[Temporal TC 08 - Users and Groups]].
- `ResourceExhausted` → per-account RPS cap hit; check [[Temporal TC 10 - Cloud Metrics]] before requesting a bump.
- Worker stuck "polling" → port 7233 blocked outbound from the Worker network.

## Operational notes
- Workers must reach the namespace endpoint outbound on TCP 7233. No inbound ports are required.
- API keys and certs both expire — Cloud sends notifications at 30/20/10 days for keys and 15/10/5 days for certs (see [[Temporal TC 14 - Cloud Operations and tcld]]).
- For HA namespaces, both regional endpoints are reachable through a single namespace endpoint that DNS-routes during failover (see [[Temporal TC 13 - High Availability]]).
- See [[Temporal SH 07 - Security]] and [[Temporal BP 08 - Security Controls]] for the broader hardening checklist.

💡 **Takeaway:** Pick mTLS or API keys, decide whether the namespace needs a public rule, and let connectivity rules enforce the answer. Workers only need outbound `:7233`, and PrivateLink/PSC are there when "off the internet" is a hard requirement.
