# 08 — Networking

🔑 Every Pod gets a routable IP (no NAT between Pods); CNI plugin makes it work; NetworkPolicy is how you actually lock it down.

Source: https://kubernetes.io/docs/concepts/services-networking/network-policies/

## The Kubernetes Network Model
1. Every Pod has a unique IP from the **Pod CIDR**.
2. Pods can talk to all other Pods without NAT.
3. Nodes can talk to all Pods without NAT.
4. The IP a Pod sees itself as = the IP others see.

The [[CNI]] plugin implements this (Calico, Cilium, Flannel, AWS VPC CNI, …).

## DNS — CoreDNS
- Default cluster DNS, runs as a [[Deployment]] in `kube-system`.
- Resolves `<svc>.<ns>.svc.cluster.local`, headless A records, and Pod records (if enabled).
- Tune `cache`, `forward`, NodeLocal DNSCache for high-QPS workloads.

## NetworkPolicy
L3/L4 allowlists. **Not enforced unless your CNI supports it** (Calico, Cilium, Antrea do).
- Default: all Pods are non-isolated. Adding *any* policy to a Pod flips it to default-deny for the selected direction.
- Rules are **additive**; no deny rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-allow-api, namespace: prod }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: api } }
      ports: [{ port: 5432, protocol: TCP }]
```

## Deny-By-Default Pattern
```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```
Then layer per-app allow rules on top.

## Service Mesh (Istio, Linkerd, Cilium)
Adds L7: mTLS between Pods, retries/timeouts, traffic shifting, distributed tracing — usually via a sidecar (Envoy) or eBPF datapath.

## ⚠️ Gotchas
- `NetworkPolicy` is namespace-scoped — cross-namespace allows need `namespaceSelector`.
- DNS resolution counts as egress; remember to allow `kube-dns` on UDP/TCP 53 in deny-all egress policies.
- 💡 `kubectl get networkpolicies -A` won't show enforcement — verify with traffic.

## Tags
[[Kubernetes]] [[CNI]] [[NetworkPolicy]] [[CoreDNS]]
