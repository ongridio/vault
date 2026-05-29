---
title: Kubernetes Pod Evicted / Node Resource Pressure
kind: howto
tags: [kubernetes, eviction, node-pressure, disk-pressure, memory-pressure, kubelet]
applies_to: [edge, manager]
---

# Kubernetes Pod Evicted / Node Resource Pressure

Use when pods show `Evicted`, or workloads get rescheduled off a node
flapping `MemoryPressure`/`DiskPressure`. The kubelet evicts pods to
reclaim a starved node resource. **The node condition + the eviction
message name the resource** — memory, ephemeral storage (disk), inodes,
or PIDs.

| Eviction reason | Starved resource |
|---|---|
| `The node was low on resource: memory` | node RAM (kubelet hit eviction threshold) |
| `... ephemeral-storage` | node disk (logs/emptyDir/images filling the FS) |
| `... inodes` | inode exhaustion on the node FS |
| `... pids` | PID limit on the node |

## Step 1 — Which node condition + which pods went

```bash
kubectl get nodes -o wide
kubectl describe node <node> | sed -n '/Conditions:/,/Addresses:/p'   # *Pressure: True?
kubectl get pods -A --field-selector status.phase=Failed | grep -i evict
kubectl describe pod <evicted-pod> | grep -A3 'Status:\|Message:'      # the reason string
```

A `True` MemoryPressure/DiskPressure condition + matching eviction
message points straight at the starved resource.

## Step 2 — Confirm on the node

```bash
# Memory
free -h; cat /proc/pressure/memory
# Disk (the kubelet watches the FS holding /var/lib/kubelet + images)
df -h /var/lib/kubelet /var/lib/containerd 2>/dev/null
df -i /var/lib/kubelet                       # inodes too
```

The kubelet's eviction thresholds (e.g. `nodefs.available<10%`) trip
before the disk is literally 100% — so a node at 92% disk can already be
evicting.

## Step 3 — Find the hog

```bash
# Disk: what's eating node ephemeral storage
du -xhd1 /var/lib/containerd /var/log 2>/dev/null | sort -h | tail
# A pod writing huge logs / emptyDir is the usual culprit
kubectl get pods -A -o wide | grep <node>
# Memory: pods without limits over-consuming
kubectl top pods -A --sort-by=memory | head
```

Disk pressure is often log spam or an unbounded `emptyDir`; memory
pressure is usually a pod with no limit (or a leak) on an
overcommitted node.

## Step 4 — Fix + prevent

- **Disk**: prune images (`crictl rmi --prune`), rotate/cap logs, bound
  `emptyDir.sizeLimit`. (Inodes → `diagnostics/inode-exhaustion.md`.)
- **Memory**: set requests/limits so the scheduler stops overpacking;
  cap the offender (it'll OOM in its own cgroup instead of starving the
  node — see `diagnostics/oom-killed.md`).
- Set sane `Guaranteed`/`Burstable` QoS so the right pods get evicted
  last.

## Decision tree

| Signal | Action |
|---|---|
| DiskPressure + log/emptyDir hog | rotate logs, bound emptyDir, prune images |
| DiskPressure + inodes 100% | `diagnostics/inode-exhaustion.md` on the node FS |
| MemoryPressure + no-limit pods | set requests/limits; cap offender |
| Evictions recur after cleanup | node undersized for the workload — scale/resize nodes |
| Best-effort pods evicted first | expected QoS behavior — set proper QoS for critical pods |

## References

- [Node-pressure Eviction — Kubernetes docs](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Configure Out-of-Resource Handling — Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#eviction-thresholds)
- vault: `diagnostics/oom-killed.md`, `diagnostics/inode-exhaustion.md`, `diagnostics/disk-pressure.md`, `systems/container/kubernetes.md`
