---
title: CPU Throttling (cgroup CFS Quota)
kind: howto
tags: [cpu, cgroup, cfs, throttling, kubernetes, limits, latency]
applies_to: [edge, manager]
---

# CPU Throttling (cgroup CFS Quota)

Use when a container/service is slow or latency-spiky **but host CPU is
not saturated** and the process isn't in D-state. The kernel's CFS
bandwidth controller hands a cgroup `quota` per 100ms `period`; burn it
early and every remaining task in that window is *frozen* until the next
period — even with idle cores on the box. This looks like random
hundred-millisecond stalls, not steady slowness.

| Symptom | Probable cause class |
|---|---|
| p99 stalls in ~100ms multiples, host CPU idle | CFS quota throttling |
| Slow only after scaling down replicas | per-pod quota too low for the load |
| Multi-threaded app throttled at low avg CPU | quota < threads × burst (parallel burn) |
| Fine on a big node, throttled on a small one | quota set absolutely, not to node size |

## Step 1 — Confirm throttling from the cgroup itself

```bash
# cgroup v2 (modern)
cat /sys/fs/cgroup/<path>/cpu.stat
#   nr_throttled / throttled_usec rising == being throttled
# cgroup v1
cat /sys/fs/cgroup/cpu/<path>/cpu.cfs_quota_us /sys/fs/cgroup/cpu/<path>/cpu.cfs_period_us
cat /sys/fs/cgroup/cpu/<path>/cpu.stat   # nr_throttled, throttled_time
```

The smoking gun is `nr_throttled` (and `throttled_usec`/`throttled_time`)
**increasing over time**. Sample twice 10s apart and diff.

## Step 2 — Quantify the throttle ratio

```bash
# Fraction of periods that hit the quota wall
#   nr_throttled / nr_periods  — anything > a few % is hurting latency
awk '/nr_periods/{p=$2} /nr_throttled/{t=$2} END{printf "throttled %.1f%%\n", 100*t/p}' \
  /sys/fs/cgroup/<path>/cpu.stat
```

In Kubernetes the same signal is the metric
`container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total`.

## Step 3 — Compare quota to actual demand

```bash
# What the limit is (k8s)
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources.limits.cpu}{"\n"}'
# What it actually uses
kubectl top pod <pod>
```

Throttled while *average* usage is well under the limit is the
tell-tale of **bursty parallel** work: 8 threads each wanting CPU for
20ms burn an 800ms-equivalent in one 100ms window against a "0.5 CPU"
(50ms) quota → throttled hard despite a low 100ms average.

## Step 4 — Fix options (in order of preference)

- **Raise the limit** (or remove the CPU *limit*, keep the *request*) so
  the scheduler still bin-packs but stops the artificial ceiling.
- **Reduce parallelism** to fit the quota (`GOMAXPROCS`, worker count,
  thread pool) — match the runtime's CPU view to the cgroup, not the
  node. (Go: set `GOMAXPROCS` to the quota; `automaxprocs` does this.)
- On older kernels, the pre-5.4 CFS slice bug over-throttled even
  compliant workloads — upgrading the kernel alone can help.

## Decision tree

| Signal | Action |
|---|---|
| `nr_throttled` rising, host idle | quota too low — raise limit or drop CPU limit |
| Throttled at low avg usage, many threads | cap parallelism / `GOMAXPROCS` to the quota |
| Only on small nodes | quota set absolutely; scale it to workload, not node |
| Throttling + old kernel (<5.4) | known CFS over-throttle bug — upgrade kernel |
| Not throttled but still slow, `b`>0 | not a quota issue — see `diagnostics/high-load-low-cpu.md` |

## References

- [CFS Bandwidth Control — kernel.org](https://www.kernel.org/doc/html/latest/scheduler/sched-bwc.html)
- [Unthrottled: How a Valid Fix Became a Kubernetes Regression — Omio](https://medium.com/omio-engineering/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718)
- [uber-go/automaxprocs](https://github.com/uber-go/automaxprocs)
- vault: `diagnostics/high-load-low-cpu.md`, `diagnostics/high-latency-p99.md`, `systems/scheduling/cfs-and-cgroups.md`
