---
title: Kubernetes Scheduler
tags: [scheduling, kubernetes, kube-scheduler]
---

# Kubernetes Scheduler

`kube-scheduler` places pending pods onto nodes. It runs a two-phase
algorithm — **filter** then **score** — over the pool of feasible nodes.
Understanding both phases is how you debug "why didn't my pod schedule".

## Filter phase

A node is filtered **out** if any of the following fail:

- **PodFitsResources** — node's allocatable >= pod's requests
- **PodFitsHostPorts** — requested host ports aren't already taken
- **PodFitsNodeSelector** — node labels satisfy `nodeSelector`
- **NodeAffinity** — `affinity.nodeAffinity` matches
- **PodAffinity / PodAntiAffinity** — co-location / spread rules satisfied
- **Taints and tolerations** — pod tolerates every NoSchedule taint
- **Volume binding** — required PVs exist and are reachable from the node

A pod stuck in `Pending` with `0/N nodes are available: M Insufficient cpu`
is failing the filter phase. The event message tells you exactly which
predicate failed.

## Score phase

Surviving nodes get ranked. Default scorers (each contributes 0–100):

- **NodeResourcesFit** — prefers more-room nodes (LeastAllocated) or
  less-room (MostAllocated, for bin-packing). Controlled by scheduler
  profile.
- **InterPodAffinity** — bonus for co-locating with affinity targets
- **NodeAffinity** — soft-affinity score
- **TaintToleration** — penalties for tolerated taints
- **ImageLocality** — bonus if the image is already pulled
- **NodeResourcesBalancedAllocation** — prefers nodes whose CPU and
  memory utilization are balanced (avoids one-resource hot spots)

The winning node gets the bind. Ties broken by node name (stable but
arbitrary).

## Requests vs. limits

The scheduler only reads `requests`. `limits` affect the kubelet's
cgroup setup (see `systems/scheduling/cfs-and-cgroups.md`) but do
**not** influence placement. Consequences:

- Set requests too low → scheduler over-packs, pods get OOM-killed
- Set requests too high → low utilization, scheduler refuses to pack
- Set limits without requests → requests default to limits, you lose
  burst capacity

Production heuristic: requests at p50-p75 of observed usage, limits at
p99 (or unset, if the workload is well-behaved).

## Taints and tolerations

A taint is `<key>=<value>:<effect>` on a node. Effects:

- `NoSchedule` — new pods without matching toleration won't schedule
- `PreferNoSchedule` — soft; scheduler tries to avoid
- `NoExecute` — already-running pods without toleration get evicted

Common operator uses:

- `node.kubernetes.io/unreachable:NoExecute` — auto-applied by node
  controller when a node goes Unknown
- Dedicated nodes for GPU / high-memory / spot — taint at provision
  time, only matching workloads land there
- Cordoning — `kubectl cordon` sets `node.kubernetes.io/unschedulable`

## Affinity vs. anti-affinity

| Construct | Meaning | Example |
|-----------|---------|---------|
| `podAffinity` | "Schedule near pods matching X" | Cache near app |
| `podAntiAffinity` | "Schedule away from pods matching X" | HA replicas across hosts |
| `topologyKey` | The dimension of "near" / "away" | `kubernetes.io/hostname` for per-host; `topology.kubernetes.io/zone` for per-zone |

**Pitfall.** `requiredDuringSchedulingIgnoredDuringExecution` is a hard
constraint at schedule time; the scheduler will leave the pod pending
forever rather than relax it. Use `preferredDuringScheduling...` unless
you genuinely need the hard version.

## Topology spread constraints

`topologySpreadConstraints` declares "spread pods of label X across
topology Y with at most Z skew". Cleaner than anti-affinity for
HA-across-zones and explicit about the skew budget.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: web
```

## Debugging stuck-pending pods

```bash
kubectl describe pod <name>            # look for Events at the bottom
kubectl get events --sort-by=.lastTimestamp
kubectl get nodes -o wide              # capacity, taints
kubectl describe node <node> | grep -A5 Taints
```

The most common stuck-pending root causes, ranked:

1. **Insufficient CPU / memory** across the cluster (requests too high)
2. **Node selector / affinity mismatch** (label typo, missing label)
3. **Taint with no toleration** (often: GPU / spot pool)
4. **PV binding failure** (zone mismatch on the PV is classic)
5. **Pod priority preemption deadlock** (low-priority pod displaced by
   a higher-priority one, then can't reschedule)
