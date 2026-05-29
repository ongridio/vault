---
title: Kubernetes PVC Pending / Volume Mount Failures
kind: howto
tags: [kubernetes, pvc, storage, volume, storageclass, mount, csi]
applies_to: [edge, manager]
---

# Kubernetes PVC Pending / Volume Mount Failures

Use when a pod is stuck `Pending` (or `ContainerCreating`) on storage: a
PVC won't bind, or a bound volume won't mount. **Separate "no volume was
ever provisioned" (PVC Pending) from "the volume exists but won't attach/
mount" (pod ContainerCreating with FailedMount).**

| Symptom | Probable cause |
|---|---|
| PVC `Pending`, no PV | no matching StorageClass / provisioner / capacity / zone |
| Pod `ContainerCreating`, `FailedAttachVolume` | volume can't attach (already attached, CSI driver, cloud quota) |
| `FailedMount` / `MountVolume.SetUp failed` | fs/permission/CSI node issue on the target node |
| `volume node affinity conflict` | PV is zone-pinned; pod scheduled to another zone |

## Step 1 — PVC bind status

```bash
kubectl get pvc <pvc>                       # STATUS Pending vs Bound
kubectl describe pvc <pvc> | sed -n '/Events:/,$p'   # provisioner error
kubectl get storageclass                    # is there a (default) class?
```

`Pending` + `no persistent volumes available for this claim and no
storage class is set` = no StorageClass / no default class. A provisioner
error in events names the real cause (quota, zone, params).

## Step 2 — Dynamic provisioning path

```bash
kubectl get pvc <pvc> -o jsonpath='{.spec.storageClassName}{"\n"}'
kubectl -n kube-system get pods | grep -iE 'csi|provisioner'   # is the provisioner running?
kubectl -n kube-system logs <csi-provisioner-pod> --tail=50
```

A dead/erroring CSI provisioner, a cloud disk quota, or an unsupported
`accessMode` (e.g. RWX on a block-only class) leaves the PVC Pending.

## Step 3 — Attach / mount failures (volume exists)

```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'   # FailedAttachVolume / FailedMount
# On the target node:
dmesg -T | grep -iE 'mount|xfs|ext4|nfs'
mount | grep <vol>
```

Common causes: a block volume still attached to a dead node
(multi-attach), a CSI node plugin not running on that node, fs corruption,
or wrong fsType. RWO volumes can't attach to two nodes — a stuck old
attachment blocks the new pod.

## Step 4 — Zone / topology affinity

```bash
kubectl get pv <pv> -o jsonpath='{.spec.nodeAffinity}{"\n"}'
kubectl get nodes -L topology.kubernetes.io/zone
```

`volume node affinity conflict` = the PV lives in zone A but the
scheduler put the pod in zone B (cloud disks are zonal). Use
`WaitForFirstConsumer` volume binding so the volume is provisioned in the
pod's zone.

## Decision tree

| Signal | Action |
|---|---|
| PVC Pending, no StorageClass | set a (default) StorageClass / provisioner |
| Provisioner error in events | fix quota/zone/params; check CSI provisioner logs |
| FailedAttach, multi-attach | detach from the dead node; RWO can't be shared |
| FailedMount on node | CSI node plugin / fsType / corruption — check dmesg |
| node affinity conflict | use WaitForFirstConsumer; schedule to the PV's zone |

## References

- [Persistent Volumes — Kubernetes docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Capacity & Volume Binding Mode — Kubernetes](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
- vault: `diagnostics/k8s-pod-stuck.md`, `diagnostics/disk-full-cn.md`, `systems/container/kubernetes.md`
