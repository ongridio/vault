---
title: OOMKilled — Process / Container Killed by the OOM Killer
kind: howto
tags: [memory, oom, oomkilled, cgroup, kubernetes, limits]
applies_to: [edge, manager]
---

# OOMKilled — Process / Container Killed by the OOM Killer

Use when a process vanished with exit 137 (128+SIGKILL), a container
shows `OOMKilled`, or a pod is CrashLoopBackOff with reason OOMKilled.
**First decide which OOM killer fired**: the per-cgroup one (the
container hit *its own* memory limit — most common) or the system one
(the *host* ran out of RAM and the kernel picked a victim). They point
at different fixes.

| Symptom | Probable cause class |
|---|---|
| One container OOMKilled, host has free RAM | hit its own cgroup limit |
| Several processes killed, host RAM exhausted | system-wide OOM (overcommit) |
| OOMKilled at startup | limit below the working set / large init allocation |
| OOMKilled hours in, climbing RSS | genuine leak — see `diagnostics/memory-leak-hunt.md` |

## Step 1 — Confirm and read the kill record

```bash
dmesg -T | grep -iE 'killed process|oom-kill|out of memory' | tail
journalctl -k --since '30 min ago' | grep -i oom
# Container exit code
docker inspect <ctr> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
```

The `oom-kill` line names the victim, its `total-vm`/`rss`, and crucially
`oom_memcg=` (the cgroup that hit its limit) vs a global kill. That field
answers the cgroup-vs-host question immediately.

## Step 2 — cgroup limit vs usage (the container case)

```bash
# cgroup v2
cat /sys/fs/cgroup/<path>/memory.max     # the limit
cat /sys/fs/cgroup/<path>/memory.current # usage at/near the kill
cat /sys/fs/cgroup/<path>/memory.events  # oom / oom_kill counters
# Kubernetes
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources.limits.memory}{"\n"}'
kubectl describe pod <pod> | grep -A3 'Last State'   # Reason: OOMKilled
```

`memory.current` slamming into `memory.max` = the limit is too low for
the real working set. Note: page cache counts toward the cgroup limit —
a process doing heavy file IO can be OOMKilled without a leak.

## Step 3 — Host-wide pressure (the system case)

```bash
free -h; cat /proc/meminfo | grep -E 'MemAvailable|Committed_AS|CommitLimit'
# Who's big right now
ps -eo pid,comm,rss --sort=-rss | head
# PSI: is memory actually under pressure?
cat /proc/pressure/memory
```

`Committed_AS` >> `CommitLimit` with default overcommit means the host
promised more than it has and the killer is the enforcement. PSI `some
avg10` > 0 for memory confirms real stall pressure.

## Step 4 — Leak vs right-sizing

Plot RSS over time (PromQL: `container_memory_working_set_bytes`). A
sawtooth that resets on restart and climbs again = leak (go to
`diagnostics/memory-leak-hunt.md`). A flat-then-killed-at-startup or a
steady plateau just above the limit = the limit is simply too small.

## Decision tree

| Signal | Action |
|---|---|
| `oom_memcg` set, host has free RAM | raise the container memory limit to fit working set |
| Killed at startup | limit below baseline — raise it / shrink init allocation |
| Page-cache-heavy, RSS modest | account for cache in the limit, or use `--memory-swap`/tune |
| Host-wide kill, `Committed_AS`>limit | reduce overcommit / add RAM / cap aggregate limits |
| RSS climbs monotonically | real leak — `diagnostics/memory-leak-hunt.md` |
| Recurs only under load | working set scales with traffic — size to peak, add headroom |

## References

- [OOM Killer — kernel.org admin guide (concepts)](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
- [Memory cgroup v2 — kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory)
- [Pressure Stall Information (PSI)](https://www.kernel.org/doc/html/latest/accounting/psi.html)
- vault: `diagnostics/memory-leak-hunt.md`, `diagnostics/memory-pressure-cn.md`, `diagnostics/swap-thrashing.md`, `systems/linux/memory.md`
