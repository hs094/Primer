# 07 — Storage

🔑 PVCs are the user-facing claim, PVs are the cluster-side resource, StorageClass dynamically provisions the PV when a claim asks for it.

Source: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

## The Three Objects
| Object | Owner | Role |
|---|---|---|
| **PersistentVolume (PV)** | Cluster | Actual storage (EBS volume, NFS export, etc.) |
| **PersistentVolumeClaim (PVC)** | Namespace | "I want 10Gi RWO." Binds 1:1 to a PV. |
| **StorageClass** | Cluster | Recipe for dynamic PV creation by a [[CSI]] driver. |

## Access Modes
| Mode | Meaning |
|---|---|
| **RWO** (ReadWriteOnce) | One node R/W. Default for block volumes (EBS, PD). |
| **ROX** (ReadOnlyMany) | Many nodes, read-only. |
| **RWX** (ReadWriteMany) | Many nodes R/W. Needs NFS / CephFS / EFS. |
| **RWOP** (ReadWriteOncePod) | Single Pod R/W (1.22+). |

## Dynamic Provisioning
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: fast-ssd }
provisioner: ebs.csi.aws.com
parameters: { type: gp3, iops: "3000" }
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

## Reclaim Policies
- **Delete** — PV + backing storage destroyed when PVC is deleted (typical for cloud).
- **Retain** — PV stays `Released`; admin must clean up. Use for irreplaceable data.

## CSI & Snapshots
Modern drivers are out-of-tree CSI plugins. Snapshots use `VolumeSnapshotClass` + `VolumeSnapshot`; restore with `dataSource: { kind: VolumeSnapshot, ... }` on a new PVC.

## ⚠️ Gotchas
- `volumeBindingMode: WaitForFirstConsumer` is almost always what you want — binds to a zone the Pod can actually run in.
- `allowVolumeExpansion: true` lets you grow PVCs; you **cannot** shrink.
- 💡 [[StatefulSet]] PVCs survive Pod deletion but **not** by default the StatefulSet — see retention policy.

## Tags
[[Kubernetes]] [[PV]] [[PVC]] [[StorageClass]] [[CSI]]
