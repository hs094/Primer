# SH 07 — Security

🔑 **Two-layer model: mTLS for transport (internode + frontend) and pluggable `Authorizer` + `ClaimMapper` for API authz. Default authorizer allows everything — set yours before any client connects.**

## Encryption in Transit (mTLS)

Temporal supports mTLS in two scopes:

| Scope | Purpose |
|---|---|
| `internode` | Encrypts traffic between Temporal Service nodes (frontend ↔ history ↔ matching ↔ worker). |
| `frontend` | Encrypts the public-facing frontend gRPC endpoint. |

Self-signed or properly minted certs both work. Example structure (full schema in [[Temporal RF 06 - Service Configuration]]):

```yaml
global:
  tls:
    internode:
      server:
        certFile: /certs/internode.pem
        keyFile: /certs/internode.key
        clientCAFiles:
          - /certs/internode-ca.pem
        requireClientAuth: true
      client:
        serverName: temporal-internode
        rootCAFiles:
          - /certs/internode-ca.pem
    frontend:
      server:
        certFile: /certs/frontend.pem
        keyFile: /certs/frontend.key
        clientCAFiles:
          - /certs/clients-ca.pem
        requireClientAuth: true
      client:
        serverName: temporal-frontend
        rootCAFiles:
          - /certs/frontend-ca.pem
```

### Server Authentication

Set `serverName` in the client section so connections validate the server certificate's CN and reject MitM.

### Client Authentication

`clientCAFiles` / `clientCAData` + `requireClientAuth: true` restricts access to clients holding certs signed by the listed CAs.

⚠️ Frontend without `requireClientAuth` is open to any TLS client — only safe behind a strict private network.

## Authorization (Two Pluggable Interfaces)

### `ClaimMapper`

Extracts `Claims` from incoming credentials (typically JWTs). Default JWT mapper accepts:

- Header: `Authorization: Bearer <jwt>`
- Permissions encoded as `<namespace>:<permission>`, where permission ∈ `read | write | worker | admin`.

Implement `GetClaims(ctx) (*Claims, error)` for custom auth (OIDC, mTLS subject mapping, SPIFFE IDs).

### `Authorizer`

`Authorize(ctx, claims, target) (Result, error)` returns `DecisionDeny` or `DecisionAllow` per RPC. Hooked into every frontend gRPC call.

### Critical Default

⚠️ **If you do not configure an `Authorizer`, Temporal uses the default `noopAuthorizer` which permits every request.** This is the single most common self-hosted security misconfiguration.

```yaml
authorization:
  jwtKeyProvider:
    keySourceURIs:
      - https://idp.example.com/.well-known/jwks.json
    refreshInterval: 1m
  permissionsClaimName: permissions
  authorizer: default
  claimMapper: default
```

For embedded servers, plug via `temporal.WithAuthorizer(...)` / `temporal.WithClaimMapper(...)` — see [[Temporal SH 03 - Embedded Server]].

## SSO for the Web UI

Configure UI separately via env vars:

```bash
TEMPORAL_AUTH_ENABLED=true
TEMPORAL_AUTH_PROVIDER_URL=https://auth.example.com
TEMPORAL_AUTH_CLIENT_ID=...
TEMPORAL_AUTH_CLIENT_SECRET=...
TEMPORAL_AUTH_CALLBACK_URL=https://temporal.example.com/auth/sso/callback
TEMPORAL_AUTH_SCOPES=openid,profile,email
```

UI auth and frontend gRPC auth are independent — protect both.

## Data Protection (Payload Codec)

Workflow inputs / results / signals are stored as raw payloads in persistence. To encrypt at rest, register a custom **Payload Codec** in your SDK clients & workers. Temporal Server never sees plaintext.

Run a **Codec Server** to let the Web UI / CLI decode payloads on demand:

- Worker / client → encode payload with key K → server stores ciphertext.
- Web UI calls Codec Server endpoint with ciphertext → decoded for display.

⚠️ Codec Server is exposed to UI users; secure it (mTLS, SSO, allowlist) or it becomes a decryption oracle.

## Network Isolation

- Run frontend on internal LB only — see [[Temporal SH 02 - Deployment]].
- Block `:7233`, history `:7234`, matching `:7235`, worker `:7239`, membership `:6933+` from public networks.
- Do not co-locate Temporal pods with internet-facing apps.

## Checklist

- [ ] mTLS enabled on **both** `internode` and `frontend`.
- [ ] `requireClientAuth: true` on frontend.
- [ ] Custom `Authorizer` + `ClaimMapper` configured (no `noopAuthorizer` in prod).
- [ ] JWT key provider points at IdP JWKS with refresh.
- [ ] UI behind SSO (`TEMPORAL_AUTH_ENABLED=true`).
- [ ] Payload codec for any sensitive data; Codec Server protected.
- [ ] No public exposure of any Temporal port. See [[Temporal BP 08 - Security Controls]].

💡 **Takeaway:** mTLS encrypts the wire; `Authorizer` + `ClaimMapper` enforce access. The default `noopAuthorizer` is a production landmine — replace it before exposure, and pair encryption-in-transit with payload codecs for at-rest protection.
