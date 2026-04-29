# 04 — Services

🔑 A Service is a stable virtual IP + DNS name in front of an ever-changing set of Pod IPs, selected by labels.

Source: https://kubernetes.io/docs/concepts/services-networking/service/

## Service Types
| Type | Reachable From | Mechanism |
|---|---|---|
| **ClusterIP** (default) | Inside cluster | Virtual IP, kube-proxy rules |
| **NodePort** | `<NodeIP>:30000-32767` | Each node opens the port |
| **LoadBalancer** | Internet | Cloud LB → NodePort → Pod |
| **ExternalName** | DNS CNAME only | No proxying, no selector |
| **Headless** (`clusterIP: None`) | Direct Pod IPs via DNS | No VIP — used by [[StatefulSet]] |

## DNS
`<svc>.<ns>.svc.cluster.local` resolves to the ClusterIP (or A records for headless).

## kube-proxy Modes
- **iptables** (default) — netfilter rules, random endpoint pick. Simple, scales to ~5k services.
- **ipvs** — kernel L4 LB, better at scale, supports rr/lc/sh algorithms.
- **nftables** (1.31+ beta) — replacement for iptables.
- Some CNIs (Cilium eBPF) replace kube-proxy entirely.

## Minimal YAML
```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: api }
  ports:
    - port: 80
      targetPort: 8080
```

## ⚠️ Gotchas
- `LoadBalancer` only provisions an LB if a cloud-controller / MetalLB is installed; on bare metal it stays `<pending>`.
- `externalTrafficPolicy: Local` preserves client IP but drops traffic on nodes with no Pod.
- 💡 Headless services are the **only** way to do client-side LB / per-Pod gRPC.

## 🧪 Try It
```bash
kubectl get endpointslices -l kubernetes.io/service-name=api
kubectl run -it --rm dns --image=busybox -- nslookup api.default
```

## Tags
[[Kubernetes]] [[Service]] [[kube-proxy]] [[CoreDNS]]
