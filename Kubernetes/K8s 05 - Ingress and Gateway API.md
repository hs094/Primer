# 05 — Ingress and Gateway API

🔑 Ingress is the legacy L7 entry point; Gateway API is its richer, role-split successor — both need a controller to do anything.

Source: https://kubernetes.io/docs/concepts/services-networking/ingress/

## Ingress
HTTP(S)-only routing into the cluster. Object is just a spec — a controller has to implement it.
- **Host + path rules** map to backend [[Service]]s.
- **TLS termination** via referenced Secret (cert + key).
- **IngressClass** picks the controller (`spec.ingressClassName: nginx`).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: web }
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
  tls: [{ hosts: [api.example.com], secretName: api-tls }]
```

## Common Controllers
| Controller | Notes |
|---|---|
| **ingress-nginx** | Reference, widely deployed. |
| **Traefik** | Dynamic config, IngressRoute CRDs. |
| **HAProxy** / **Contour** | High perf. |
| **AWS ALB / GCE / Azure AGIC** | Cloud-managed. |
| **Istio Gateway** / **Cilium Gateway** | Service mesh ingress. |

## Gateway API (GA in v1.29+)
Replacement for Ingress. Three role-separated resources:
- **GatewayClass** — installed by the platform (controller behind it).
- **Gateway** — listener config (ports, TLS), owned by infra/platform team.
- **HTTPRoute** / **TCPRoute** / **GRPCRoute** — owned by app team, attaches to a Gateway.

## ⚠️ Gotchas
- The Ingress API is **frozen** — new features land in Gateway API.
- TLS Secret must live in the **same namespace** as the Ingress (or use a controller-specific cross-ns mechanism).
- 💡 `pathType: Prefix` matches segment-wise; `/foo` matches `/foo` and `/foo/bar`, not `/foobar`.

## Tags
[[Kubernetes]] [[Ingress]] [[Gateway API]] [[TLS]]
