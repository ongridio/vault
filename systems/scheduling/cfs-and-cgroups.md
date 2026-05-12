---
title: CFS and cgroups
tags: [scheduling, cfs, cgroups, cpu]
---

# CFS and cgroups

The CPU side of Linux scheduling is **CFS** (Completely Fair Scheduler).
The resource-isolation side is **cgroups v2**. They cooperate: cgroups
declare entitlements; CFS enforces them within a scheduling class.

## CFS in one paragraph

CFS picks the runnable task with the smallest *virtual runtime*
(`vruntime`) so far, runs it for a slice, then re-queues it with updated
`vruntime`. Nice values weight `vruntime` accrual — a nice +10 task
accrues vruntime ~10x faster, so CFS picks it ~10x less often. There are
no fixed time slices; the slice shrinks as more runnable tasks compete.

Implication: **CPU "fair share" emerges from vruntime accounting**, not
from quotas. CFS alone can't enforce "this group gets exactly 30%" —
that's what cgroups add.

## cgroups v2 CPU controllers

Two knobs cohabit:

### `cpu.weight` (relative share)

Range 1–10000, default 100. Larger weight = larger share when there's
contention. **Only matters when CPU is saturated.** With idle CPU, every
group runs as much as it wants.

```
echo 200 > /sys/fs/cgroup/<group>/cpu.weight   # 2x default share
```

### `cpu.max` (hard cap)

Two numbers: `<max> <period>` in microseconds. The group is allowed
`<max>` microseconds of CPU time per `<period>` microseconds. Setting
`50000 100000` means "at most 50% of one CPU on average, regardless of
idle capacity elsewhere."

```
echo "50000 100000" > /sys/fs/cgroup/<group>/cpu.max
```

**Hard caps cause throttling.** A burst beyond the cap gets paused
until the next period. `cpu.stat` shows `nr_throttled` and
`throttled_usec`. Latency-sensitive workloads should prefer `cpu.weight`
unless you genuinely need a noisy-neighbor firewall.

### Pitfall — multi-threaded throttling

A 16-vCPU host with a 4-CPU quota means the group can use 4 CPU-seconds
per period across all its threads. A burst of parallel work on all 16
vCPUs drains the quota in 100ms/(16/4) = 25ms, and the group is
throttled for the rest of the 100ms period. Effect: tail latency
explodes even though *average* CPU is under the cap.

Mitigation: raise `cpu.cfs_period_us` (longer period = less aggressive
throttling), or move to `cpu.weight`.

## Other cgroup controllers worth knowing

| Controller | What it bounds | Notable knob |
|------------|---------------|--------------|
| memory     | RSS + page cache | `memory.max` (OOM kill on overrun), `memory.swap.max` |
| io         | block IO | `io.weight` (cfq-style), `io.max` (bps + iops cap) |
| pids       | task count | `pids.max` (fork bomb defense) |
| cpuset     | which CPUs/nodes | `cpuset.cpus`, `cpuset.mems` (NUMA pinning) |

`cpu.max` and `memory.max` are the two that most often cause confusing
production behavior; learn their failure modes before relying on them.

## How container runtimes use these

Both Docker and Kubernetes ultimately write to cgroup files:

- Docker `--cpus 0.5` → `cpu.max 50000 100000`
- Docker `--cpu-shares 512` → `cpu.weight ≈ 50` (default share is 1024
  on legacy, 100 on cgroups v2)
- K8s `resources.requests.cpu: 500m` → `cpu.weight` (proportional)
- K8s `resources.limits.cpu: 1` → `cpu.max 100000 100000`

This is why the K8s "requests vs. limits" distinction matters so much:
requests influence *scheduling decisions* and *cpu.weight*; limits add
hard caps via *cpu.max*.

## Diagnosing CPU contention

```bash
# Per-cgroup CPU stat (cgroup v2)
cat /sys/fs/cgroup/<group>/cpu.stat
# nr_throttled / throttled_usec / nr_bursts

# Run-queue length per CPU
mpstat -P ALL 1 5

# Process-level CPU
pidstat -u 1 5
```

High `nr_throttled` on a group whose `cpu.max` is set is the classic
"latency mystery" — the average load on the host looks fine.
