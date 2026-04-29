# 09 — RBAC

🔑 RBAC is purely additive (no deny rules). Bind a Role to a subject (user, group, or ServiceAccount) and that's the entire permission set — least privilege is on you.

Source: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

## Four Objects
| Object | Scope | What it does |
|---|---|---|
| **Role** | Namespace | Verbs on resources (e.g. `get pods`) |
| **ClusterRole** | Cluster | Same, but cluster-wide or for cluster-scoped resources (nodes, PVs) |
| **RoleBinding** | Namespace | Binds a Role *or* ClusterRole to subjects, scoped to that namespace |
| **ClusterRoleBinding** | Cluster | Binds a ClusterRole to subjects across all namespaces |

## Role Example
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: prod, name: pod-reader }
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

## Binding to a ServiceAccount
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { namespace: prod, name: api-reads-pods }
subjects:
  - { kind: ServiceAccount, name: api, namespace: prod }
roleRef:
  { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```

## ServiceAccounts
Pods authenticate to the apiserver as a ServiceAccount (default: `default` in their namespace). Token is mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` (projected, time-bound in 1.22+).

## Useful kubectl
```bash
kubectl auth can-i list secrets -n prod
kubectl auth can-i '*' '*' --as=system:serviceaccount:prod:api
kubectl auth whoami
```

## ⚠️ Gotchas
- `cluster-admin` is a wildcard — never bind it to a workload SA.
- Aggregated ClusterRoles (`aggregationRule`) silently pull in rules from labeled ClusterRoles — audit them.
- 💡 Privilege escalation is blocked — you can't grant a Role with verbs you don't already have. (`escalate` verb required to bypass.)

## Tags
[[Kubernetes]] [[RBAC]] [[ServiceAccount]]
