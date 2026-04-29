# 01 — Architecture

🔑 A cluster is a control plane (brains) + nodes (muscle), wired together by three pluggable interfaces: CRI (runtime), CNI (network), CSI (storage).

Source: https://kubernetes.io/docs/concepts/architecture/

## Control Plane
| Component | Role |
|---|---|
| **kube-apiserver** | REST front door; everything goes through it. Horizontally scalable. |
| **etcd** | Consistent, HA key-value store; the only source of cluster truth. Back it up. |
| **kube-scheduler** | Picks a node for each unassigned [[Pod]] using requests, affinity, taints. |
| **kube-controller-manager** | Runs reconcile loops: Node, Job, EndpointSlice, ServiceAccount, etc. |
| **cloud-controller-manager** | Talks to the cloud provider (LBs, routes, node lifecycle). Optional. |

## Node Components
- **kubelet** — agent on every node; ensures containers in PodSpecs are running and healthy.
- **kube-proxy** — programs iptables/IPVS so [[Service]] VIPs reach Pods. Some CNIs (Cilium) replace it.
- **container runtime** — containerd / CRI-O / etc. Speaks CRI to the kubelet.

## Plugin Interfaces
- **CRI** — kubelet ↔ runtime. Decoupled containerd/CRI-O from in-tree code.
- **[[CNI]]** — pod networking + IPAM. Calico, Cilium, Flannel, Weave.
- **[[CSI]]** — out-of-tree storage drivers (EBS, Ceph, NFS). Replaces in-tree provisioners.

## ⚠️ Gotchas
- Lose etcd quorum → cluster is read-only at best. Run 3 or 5 etcd members, never 2 or 4.
- The apiserver is the only component that writes to etcd; controllers and kubelets just call it.
- 💡 `kubectl get componentstatuses` is deprecated; check control plane Pods in `kube-system` instead.

## 🧪 Try It
```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods
kubectl get --raw='/readyz?verbose'
```

## Tags
[[Kubernetes]] [[etcd]] [[CNI]] [[CSI]] [[kubelet]]
