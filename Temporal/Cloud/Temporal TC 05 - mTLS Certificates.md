# TC 05 — mTLS Certificates

🔑 **Upload your CA bundle, hand workers an end-entity cert + key, Cloud verifies on every connection. No shared secrets — only the public cert is exchanged.**

## Why mTLS

Use mTLS over [[Temporal TC 04 - API Keys]] when:
- You already run a private CA / PKI
- You want transport-layer identity, not bearer tokens
- Compliance requires cert-based mutual auth
- You want certificate filters to restrict which leaf certs can connect to which namespace

## CA Certificate Requirements

Bundle PEM you upload to a Namespace must satisfy:

- **X.509v3** format
- **Each cert** is either a self-signed root or signed by another cert in the bundle
- **`CA: true`** in basic constraints
- **RSA or ECDSA**, signed with **SHA-256 or stronger** (no SHA-1, no MD5)
- **No passphrase** on the private key (irrelevant to upload, but you'll need keyless when signing leaf certs)
- **Bundle limits:** ≤16 certificates, ≤32 KB before base64 encoding
- **Unique Distinguished Names** across the chain (DN comparison is case-insensitive)

## End-Entity (Client) Certificate Requirements

What workers/clients present:

- X.509v3
- `CA: false`
- Digital Signature key usage
- RSA or ECDSA, SHA-256 or stronger
- Issued by a cert in the uploaded CA bundle (directly or transitively)

⚠️ **Don't use public CAs (DigiCert, Let's Encrypt, etc.)** unless you also configure certificate filters. Otherwise *anyone* with a leaf cert from that CA could connect.

## Generating + Uploading Certs

With `tcld` (no existing PKI):

```bash
tcld gen ca   --org acme -d 1y   --ca-cert ca.pem --ca-key ca.key
tcld gen leaf --org acme -d 364d --ca-cert ca.pem --ca-key ca.key \
              --cert client.pem --key client.key
```

`step` CLI works equivalently. Java SDK needs PKCS8: `openssl pkcs8 -topk8 -inform PEM -outform PEM -in client.key -out client.pkcs8.key -nocrypt`.

Upload — UI: Namespace → Edit → **Authentication** → paste PEM. CLI:

```bash
tcld namespace accepted-client-ca set \
  --namespace payments-prod.abc12 \
  --ca-certificate-file ca.pem
```

## Connecting a Worker

```bash
# Temporal CLI
temporal workflow list \
  --address payments-prod.abc12.tmprl.cloud:7233 \
  --namespace payments-prod.abc12 \
  --tls-cert-path client.pem --tls-key-path client.key
```

All SDKs (Go, Java, Python, TypeScript, .NET, PHP, Ruby) support mTLS — each takes cert path/bytes + key path/bytes. Java SDK needs PKCS8 keys (`openssl pkcs8 -topk8`).

## Zero-Downtime CA Rotation

The rollover pattern — keep old + new CAs trusted, then drop the old:

```bash
# 1. Combined bundle
cat ca-old.pem ca-new.pem > ca-rollover.pem
tcld namespace accepted-client-ca set --namespace payments-prod.abc12 --ca-certificate-file ca-rollover.pem
# 2. Reissue worker leaf certs from new CA, roll workers
# 3. Confirm no traffic on old CA via metrics
# 4. Push new-only bundle
tcld namespace accepted-client-ca set --namespace payments-prod.abc12 --ca-certificate-file ca-new.pem
```

UI equivalent: Edit → Authentication → append new PEM → save → wait → remove old.

## Certificate Filters

Restrict which leaf certs (out of those signed by your CA) can authenticate. Match on **CN**, **O**, **OU**, or **SAN**.

- Case-insensitive
- Wildcard `*` at start *or* end only — never middle, never alone
- Up to **25 filters per namespace**; specified fields must *all* match

```
✅  *.payments.acme.com    ✅  Acme Payments*
❌  acme.*.com             ❌  *
```

```bash
tcld namespace certificate-filters import --namespace payments-prod.abc12 --input-file filters.json
tcld namespace certificate-filters export --namespace payments-prod.abc12
tcld namespace certificate-filters clear  --namespace payments-prod.abc12
```

Filters let one CA serve many namespaces with per-NS access enforcement.

## Compromise Response

⚠️ **Temporal does not check CRLs or OCSP.** No server-side revocation. Mitigation:

1. Add a cert filter that excludes the compromised leaf's CN/SAN
2. Generate new CA, run the rotation pattern above
3. Reissue all leaf certs under the new CA
4. Drop the compromised CA from the bundle
5. Audit logs for unauthorized actions during the exposure window

Short-lived leaves (days, not years) bound the blast radius.

## Expiration

⚠️ **Expired root CA invalidates every leaf** — entire fleet drops at once.

⚠️ **Expired leaf** = one worker offline. Cloud emails **15 days** ahead but don't rely on it; automate reissuance.

## Authorization Models

| Model | When |
|---|---|
| Separate root CA per namespace | Strict isolation, more PKI overhead |
| Single root CA + cert filters | Many namespaces, central PKI |

⚠️ **Namespace Admin** required to manage CA bundle and filters — [[Temporal TC 06 - Account Access]], [[Temporal TC 07 - Roles and Permissions]]. Self-hosted contrast: [[Temporal SH 07 - Security]].

💡 **Takeaway:** Upload a CA bundle, hand workers leaf certs, rotate with a combined-bundle window. No CRL — use cert filters and short-lived leaves to bound compromise. Two-CA rollover is your zero-downtime primitive; build it into PKI runbooks before you need it.
