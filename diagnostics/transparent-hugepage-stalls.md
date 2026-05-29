---
title: Transparent Huge Pages (THP) Latency Stalls
kind: howto
tags: [memory, thp, hugepages, latency, khugepaged, compaction, jitter]
applies_to: [edge, manager]
---

# Transparent Huge Pages (THP) Latency Stalls

Use when a latency-sensitive service (database, low-latency app) has
mysterious jitter / periodic pauses with no GC or IO cause, on a host
where THP is `always`. THP coalesces 4K pages into 2M pages to cut TLB
misses — but **synchronous compaction to form a huge page can stall an
allocating thread**, and `khugepaged` churns in the background. Many DBs
explicitly recommend disabling THP for this reason.

| Symptom | Probable cause |
|---|---|
| Periodic latency spikes, no GC/IO cause | direct compaction stall forming a THP |
| `khugepaged` using CPU continuously | background THP collapse churn |
| DB docs say "disable THP" and it's `always` | known THP latency interaction |
| High `compact_stall` in vmstat | allocations blocking on compaction |

## Step 1 — THP mode + activity

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled   # [always] madvise never
cat /sys/kernel/mm/transparent_hugepage/defrag     # defrag policy
grep -E 'thp_|compact_stall|compact_fail' /proc/vmstat
```

`[always]` + rising `compact_stall` = allocations are paying for
synchronous compaction. `thp_collapse_alloc` churn reflects `khugepaged`
activity.

## Step 2 — Correlate stalls with compaction

```bash
# khugepaged CPU
top -b -n1 | grep khugepaged
# Are huge pages actually being used by the process?
grep -i AnonHugePages /proc/<pid>/smaps_rollup 2>/dev/null
```

If latency spikes line up with `compact_stall` increments or `khugepaged`
activity, THP is your jitter source.

## Step 3 — The standard fix for latency-sensitive services

```bash
# Disable THP globally (or set madvise so only opt-in apps get it)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# Persist via kernel cmdline (transparent_hugepage=never) or a tuned/systemd unit
```

`madvise` is the middle ground: THP only for apps that explicitly ask
(`madvise(MADV_HUGEPAGE)`), avoiding surprise stalls for everyone else.
Databases (Mongo, Redis, Oracle, many JVMs) recommend `never` outright.

## Step 4 — When THP helps (don't blanket-disable)

THP genuinely reduces TLB pressure for big-heap, throughput-oriented,
sequential-access workloads. Disable it for **latency**-sensitive
services; leave it (or use `madvise`) for throughput workloads that
benefit. Measure, don't cargo-cult.

## Decision tree

| Signal | Action |
|---|---|
| latency spikes + compact_stall rising | set THP `never` (or `madvise`) for the service |
| khugepaged hot | THP collapse churn — `never`/`madvise` |
| DB recommends disabling THP | follow it — set `never`, persist on boot |
| throughput app, no latency issue | leave THP on (or `madvise`) — it may help |

## References

- [Transparent Hugepage Support — kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html)
- [Redis — THP latency warning](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/#transparent-huge-pages)
- vault: `diagnostics/high-latency-p99.md`, `diagnostics/page-cache-pressure.md`, `systems/linux/memory.md`
