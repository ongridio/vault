---
title: Swap Thrashing / High Swap Usage
kind: howto
tags: [memory, swap, thrashing, psi, paging, performance]
applies_to: [edge, manager]
---

# Swap Thrashing / High Swap Usage

Use when the `swap_high` alert fires, or a box is mysteriously sluggish
with high iowait but disks aren't busy with real work. **High swap
*usage* is not automatically a problem — high swap *activity* (paging in
and out continuously) is.** A box can sit with 50% swap full and be
perfectly healthy if nothing is actively being paged. Measure the rate,
not the level, first.

| Symptom | Probable cause class |
|---|---|
| Swap full, `si/so` ≈ 0, system fine | cold pages parked — benign, leave it |
| `si/so` continuously high, iowait up | active thrashing — memory genuinely short |
| Latency cliffs when a service is touched | its pages were swapped out, fault-in on access |
| Swap climbing steadily over days | slow leak pushing the working set out |

## Step 1 — Level vs activity

```bash
free -h                              # how much swap is used (level)
vmstat 1 5                           # si (swap-in) / so (swap-out) per sec (activity)
cat /proc/pressure/memory            # PSI: real stall pressure?
```

The verdict:
- `si`/`so` near 0 → swap is just holding cold pages, **not your
  problem** — don't panic-disable swap.
- `si`/`so` sustained > 0 (esp. both) → active paging. PSI memory
  `some avg10 > 0` confirms tasks are stalling on memory.

## Step 2 — Who is consuming memory (and who got swapped)

```bash
# Top RSS consumers
ps -eo pid,comm,rss,%mem --sort=-rss | head
# Per-process swap footprint
for f in /proc/*/status; do awk '/^VmSwap/{s=$2} /^Name/{n=$2} END{if(s>0) print s" kB "n}' "$f"; done | sort -rn | head
# Or: smem -t -k -c "pid name swap" 2>/dev/null
```

This tells you whether the swapped-out pages belong to the hot service
(bad — it'll fault them back in and stall) or to idle daemons (fine).

## Step 3 — Is it real shortage or reclaimable?

```bash
cat /proc/meminfo | grep -E 'MemAvailable|Cached|Buffers|Dirty|SReclaimable'
```

`MemAvailable` is the honest "how much can I get without swapping"
number — large `Cached`/`SReclaimable` means the kernel can reclaim
instead of swap, so true shortage is smaller than `free` first suggests.

## Step 4 — Tunables and structural fixes

```bash
sysctl vm.swappiness                 # 60 default; lower = prefer reclaim over swap
# Lower swappiness reduces anonymous-page swapping under cache pressure:
sysctl -w vm.swappiness=10
```

But swappiness only changes the *preference* — if the working set truly
exceeds RAM, the real fixes are: add RAM, reduce per-process footprint,
cap cgroup memory so the offender is contained (or OOMKilled rather than
dragging the whole host — see `diagnostics/oom-killed.md`). On dedicated
latency-critical nodes some operators disable swap entirely; weigh that
against losing the safety valve.

## Decision tree

| Signal | Action |
|---|---|
| Swap full but `si/so`≈0, PSI quiet | benign — no action, do not disable swap reflexively |
| Sustained `si/so`, PSI stalling | real shortage — add RAM / shrink footprint / cap cgroups |
| Hot service's pages swapped out | lock/raise its memory; lower swappiness; isolate noisy neighbor |
| Steady multi-day climb | leak pushing working set out — `diagnostics/memory-leak-hunt.md` |
| Cache large, MemAvailable healthy | reclaimable, not a true shortage — investigate the alert threshold |

## References

- [In defence of swap — Chris Down](https://chrisdown.name/2018/01/02/in-defence-of-swap.html)
- [Pressure Stall Information (PSI)](https://www.kernel.org/doc/html/latest/accounting/psi.html)
- [vm.swappiness and the page cache — kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html)
- vault: `diagnostics/oom-killed.md`, `diagnostics/memory-pressure-cn.md`, `diagnostics/memory-leak-hunt.md`, `systems/linux/memory.md`, `alerts/host-metrics.md`
