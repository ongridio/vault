---
title: Context-Switch Storm / Scheduler Thrash
kind: howto
tags: [cpu, scheduler, context-switch, runqueue, threads, contention]
applies_to: [edge, manager]
---

# Context-Switch Storm / Scheduler Thrash

Use when CPU is busy in `%sys` (kernel), load is high, but the app does
little useful work — and `vmstat`'s `cs` (context switches/sec) is huge.
Excessive switching means threads aren't running long enough to make
progress: too many runnable threads, lock contention bouncing threads, or
interrupt/wakeup storms. **Separate voluntary (waiting on locks/IO) from
involuntary (preempted — too many threads for cores).**

| Symptom | Probable cause |
|---|---|
| High `cs`, high `%sys`, low useful work | thread over-subscription / lock contention |
| Mostly involuntary switches | more runnable threads than cores (preemption) |
| Mostly voluntary switches | lock/IO waits — threads blocking constantly |
| `cs` spikes with wakeups | thundering herd / timer/IRQ storm |

## Step 1 — Quantify switching + run queue

```bash
vmstat 1 5                            # cs = ctx switches/sec; r = runnable threads
pidstat -w 1 5                        # cswch/s (voluntary) + nvcswch/s (involuntary) per proc
mpstat -P ALL 1 3                     # is it %sys-heavy?
```

`r` (runqueue) >> cores with high involuntary switches = over-subscription.
`pidstat -w` localizes which process is switching most and whether it's
voluntary (blocking) or involuntary (preempted).

## Step 2 — Voluntary: lock / IO contention

High `cswch/s` (voluntary) means threads block and yield constantly —
classic lock contention. Profile where they block:

```bash
# Off-CPU / wait points (perf, if available)
perf sched record -- sleep 5; perf sched latency 2>/dev/null | head
cat /proc/<pid>/status | grep -i ctxt        # cumulative switch counts
```

A mutex everyone fights over serializes work and ping-pongs threads —
the fix is in the app (reduce lock scope, shard the lock, lock-free).

## Step 3 — Involuntary: too many threads

High `nvcswch/s` (involuntary) means the scheduler preempts threads
because there are more runnable than cores. A thread pool / `GOMAXPROCS`
/ worker count far exceeding cores causes constant preemption with little
gain. Size pools to cores (and to the cgroup quota — see
`diagnostics/cpu-throttling.md`).

## Step 4 — Wakeup / IRQ storms

A flood of interrupts or timer wakeups drives switching independent of
the app. Check `/proc/interrupts` and softirqs (see
`diagnostics/softirq-ksoftirqd-high.md`); a chatty NIC or a high-res timer
abuser can dominate.

## Decision tree

| Signal | Action |
|---|---|
| high voluntary cswch | lock/IO contention — reduce lock scope, profile waits |
| high involuntary nvcswch | too many threads — size pools to cores/quota |
| `r` >> cores | over-subscription — cap concurrency |
| switching tracks IRQs/wakeups | IRQ/timer storm — see softirq playbook |
| `%sys` high, syscall-heavy | reduce syscall rate / batching in the app |

## References

- [Brendan Gregg — Linux Performance (scheduler)](https://www.brendangregg.com/linuxperf.html)
- [pidstat(1) — man7](https://man7.org/linux/man-pages/man1/pidstat.1.html)
- vault: `diagnostics/cpu-throttling.md`, `diagnostics/high-load-low-cpu.md`, `diagnostics/softirq-ksoftirqd-high.md`, `systems/scheduling/cfs-and-cgroups.md`
