# Kubernetes — INDEX

🔑 Master index for the [[Kubernetes]] knowledge pack. Sourced from https://kubernetes.io/docs/.

## Foundations
- [[K8s 01 - Architecture]] — control plane, nodes, CRI/CNI/CSI.
- [[K8s 02 - Pods]] — smallest unit, multi-container, lifecycle.
- [[K8s 03 - Workloads]] — Deployment, StatefulSet, DaemonSet, Job, CronJob.

## Networking
- [[K8s 04 - Services]] — ClusterIP/NodePort/LoadBalancer/headless, kube-proxy.
- [[K8s 05 - Ingress and Gateway API]] — L7 routing, TLS, controllers.
- [[K8s 08 - Networking]] — CNI, Pod CIDR, NetworkPolicy, DNS, mesh.

## Configuration & Storage
- [[K8s 06 - Config and Secrets]] — ConfigMap, Secret, ESO, Vault.
- [[K8s 07 - Storage]] — PV/PVC, StorageClass, CSI, snapshots.

## Security
- [[K8s 09 - RBAC]] — Role, ClusterRole, ServiceAccount, least privilege.

## Health & Resources
- [[K8s 10 - Probes and Health]] — liveness/readiness/startup, graceful shutdown.
- [[K8s 11 - Resource Limits]] — requests vs limits, QoS, ResourceQuota.
- [[K8s 12 - Autoscaling]] — HPA, VPA, Cluster Autoscaler, KEDA, Karpenter.

## Packaging & Extension
- [[K8s 13 - Helm]] — chart structure, templating, releases, OCI.
- [[K8s 14 - Kustomize]] — overlays, patches, generators.
- [[K8s 15 - Operators and CRDs]] — CRDs, reconcile loop, frameworks.

## Operations
- [[K8s 16 - Observability and Debug]] — kubectl, describe/logs/exec, metrics-server, Prometheus.

## Cross-Vault
- [[Docker]] — container basics under [[Kubernetes]].
- [[Helm]] · [[Kustomize]] · [[CRD]] · [[Operator]] · [[CNI]] · [[CSI]]
- Adjacent: [[Observability]] · [[NATS]] · [[Kafka]] · [[PostgreSQL]] (often deployed via [[Operator]]s).

## Tags
[[Kubernetes]] [[INDEX]]
