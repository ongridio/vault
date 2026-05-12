---
title: Linux Memory Model
tags: [linux, memory]
---

# Linux Memory Model

The thing that trips most operators up: Linux uses RAM aggressively for
the page cache, so `free` showing "almost no free memory" is usually
normal. What you actually care about is `MemAvailable`.

## The numbers that matter

```
$ cat /proc/meminfo
MemTotal:     16273172 kB    # all RAM the kernel sees
MemFree:        421596 kB    # genuinely unused — usually small, that's fine
MemAvailable: 11842184 kB    # what a new allocation can claim — THIS is the one
Cached:       9123012 kB    # page cache + tmpfs
Buffers:        38152 kB    # block-device buffer cache
Slab:          624188 kB    # kernel object cache
SReclaimable:  421984 kB    # slab the kernel will give back under pressure
```

**Rule of thumb.** `MemAvailable / MemTotal` < 10% is the threshold for
worry; `< 5%` is the threshold for action.

## Page cache vs. anonymous

- **Page cache** (file-backed) is reclaimable — the kernel can drop it
  any time and refetch from disk
- **Anonymous memory** (heap, stack) is only reclaimable via swap

A growing `Cached` is almost always fine. A growing
`AnonPages` + flat `Cached` is what a leak looks like.

## Swap behavior

`vm.swappiness` (0–100) controls swap aggressiveness:

- `60` (default) — balanced
- `10` — favor RAM, only swap under pressure (good for latency-sensitive
  workloads with cold pages)
- `100` — aggressive swap, OK for batch workloads that tolerate IO

`vmstat 1` showing sustained `si`/`so` (swap in/out) means active
thrashing. A small steady non-zero `so` shortly after a memory pressure
event is normal.

## OOM killer

Triggers when `MemAvailable` hits ~0 and the kernel can't reclaim fast
enough. Picks a victim by `oom_score` (size + nice value + adjustments).

Check post-mortem:

```bash
dmesg -T | grep -i 'killed process'
journalctl -k | grep -i oom
```

Protect critical processes:

```bash
echo -1000 > /proc/$PID/oom_score_adj   # never kill
echo  500  > /proc/$PID/oom_score_adj   # prefer killing this one
```

## When the alert says `mem_high`

Walk through: `MemAvailable` first, then `vmstat 1` for swap activity,
then `ps aux --sort=-rss | head` to find the consumer. See the
[host-metrics alert runbook](../../alerts/host-metrics.md#mem_high).
