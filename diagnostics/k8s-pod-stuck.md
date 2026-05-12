---
title: K8s — Pod Stuck in Pending / CrashLoopBackOff
tags: [kubernetes, diagnostics, pod]
---

# K8s — Pod Stuck in Pending / CrashLoopBackOff

Branch by state first. The signals are different.

## Pending

`kubectl describe pod <name>` — read the `Events:` at the bottom.

### `0/N nodes are available: M Insufficient cpu/memory`

Cluster doesn't have a node with enough free `requests` capacity.

```bash
kubectl get nodes -o wide
kubectl describe nodes | grep -A6 "Allocated resources"
kubectl get pods -A --field-selector=status.phase=Running -o json \
  | jq -r '.items[].spec.containers[].resources.requests // empty'
```

Decide:

- **Workload requests too high** → tune requests to p75 of observed
  usage and re-apply
- **Cluster genuinely full** → scale the node pool, or evict
  lower-priority pods (set up PriorityClass if you haven't)

### `0/N nodes are available: M node(s) had untolerated taint`

Pod doesn't tolerate a node taint.

```bash
kubectl describe node <name> | grep -A2 Taints
```

Either add the toleration to the pod, or pick a different node pool.

### `0/N nodes are available: M node(s) didn't match Pod's node affinity/selector`

Label / affinity rules eliminate all nodes.

```bash
kubectl get nodes --show-labels
kubectl get pod <name> -o yaml | yq '.spec | .nodeSelector, .affinity'
```

Common typo: label `disktype: ssd` on nodes vs. `nodeSelector:
disk-type: ssd` on the pod.

### PV / PVC binding

```bash
kubectl get pvc <name>
kubectl get pv
kubectl describe pvc <name>
```

Stuck reasons:

- **StorageClass missing or default not set** — `kubectl get sc`
- **Provisioner unavailable** — CSI driver pod down; check the
  csi-driver namespace
- **Zone mismatch** — PV is in zone-a, pod scheduled to zone-b. Add a
  `volumeBindingMode: WaitForFirstConsumer` on the StorageClass

## ContainerCreating (sub-state of Pending)

`describe` shows reason like `FailedCreatePodSandBox` or
`FailedMount`.

```bash
kubectl describe pod <name>
```

Walk these:

- **Image pull** — `ErrImagePull` / `ImagePullBackOff` → registry creds,
  image tag exists, network to the registry
- **Mount failure** — CSI / NFS unavailable, secret / configmap
  referenced but missing
- **CNI failure** — node out of IPs, or CNI plugin broken; check
  `journalctl -u kubelet` and the CNI daemonset

## CrashLoopBackOff

Container starts, exits non-zero, kubelet restarts it with exponential
backoff (10s → 20s → ... capped at 5min).

```bash
kubectl logs <pod> --previous           # ← last terminated container
kubectl logs <pod> --all-containers --previous
kubectl describe pod <pod>              # exit code, last state reason
```

Read the **last 30 lines** of the previous container's stdout — the
panic / fatal log is almost always there.

Common causes:

- **Bad config** — env var missing, configmap path wrong, secret
  not mounted yet (race with init container)
- **Liveness probe killing a slow starter** — add a `startupProbe` or
  raise `initialDelaySeconds`
- **OOMKilled** — `kubectl get pod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'`
  shows `OOMKilled`. Bump `memory.limits` or fix the leak.
- **Missing dependency at boot** — DB not reachable yet; use an
  initContainer to gate startup

## Terminating stuck

Pod stays in `Terminating` past the grace period.

```bash
kubectl get pod <name> -o yaml | grep -A2 finalizers
```

If a finalizer is listed, some controller is holding the pod and
should remove the finalizer when its cleanup is done. Force-delete
only when you've confirmed nothing is actually pending:

```bash
kubectl delete pod <name> --grace-period=0 --force
```

Then investigate why the finalizer was stuck (most often: the
controller managing it is down, or a custom CRD's reconcile loop
crashed).

## High-leverage commands

```bash
kubectl get events --sort-by=.lastTimestamp -A | tail -30
kubectl describe pod <name>
kubectl logs <pod> -c <container> --previous
kubectl get pod <name> -o yaml
kubectl debug node/<node> -it --image=busybox    # node-level shell
```

## See also

- `systems/scheduling/kubernetes-scheduler.md` — filter / score
- `systems/container/kubernetes.md` — overall model
