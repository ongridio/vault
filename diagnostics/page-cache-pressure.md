---
title: Page Cache Pressure & Reclaim Stalls
kind: howto
tags: [memory, page-cache, reclaim, psi, vfs, io]
applies_to: [edge, manager]
---

# Page Cache Pressure & Reclaim Stalls

Use when "free" memory looks alarmingly low but the box is fine — or when
it *isn't* fine because reclaiming cache to satisfy allocations is
stalling the app. The kernel uses all spare RAM as page cache (good); the
question is whether reclaim is cheap (drop clean cache) or expensive
(write back dirty pages, swap). **`free` low is normal; reclaim *stalls*
are the real problem — measure with PSI.**

| Symptom | Probable cause |
|---|---|
| `free` near zero, lots of `Cached`, fine | normal — cache uses spare RAM, reclaimable |
| Latency stalls + memory PSI > 0 | direct reclaim blocking allocations |
| Heavy IO evicts a hot working set | cache thrash — working set > RAM |
| `kswapd` busy continuously | sustained reclaim pressure |

## Step 1 — Read memory honestly

```bash
free -h                              # used vs available (not "free"!)
cat /proc/meminfo | grep -E 'MemAvailable|Cached|Dirty|Writeback|SReclaimable'
```

`MemAvailable` is the real "can allocate without painful reclaim" number.
Large `Cached` + healthy `MemAvailable` = the low "free" is fine.

## Step 2 — Is reclaim actually stalling?

```bash
cat /proc/pressure/memory            # some/full avg10 — stall time %
# kswapd (background reclaim) vs direct reclaim (app blocks)
grep -E 'pgscan|pgsteal|allocstall' /proc/vmstat
```

Memory PSI `some avg10 > 0` = tasks are stalling on memory reclaim.
Rising `allocstall*` = **direct reclaim**: an allocating thread had to
reclaim synchronously (blocking) because `kswapd` couldn't keep up — that's
the latency you feel.

## Step 3 — Cache thrash (working set > RAM)

If a hot dataset keeps getting evicted then re-read from disk (IO climbs,
cache hit rate drops), the working set exceeds RAM. Confirm with rising
major faults / IO. The fix is more RAM, a smaller working set, or
accepting the IO — not a sysctl.

## Step 4 — Tunables (use sparingly)

```bash
sysctl vm.swappiness                 # lower = reclaim cache before swapping anon
sysctl vm.vfs_cache_pressure         # >100 = reclaim dentries/inodes more aggressively
# Don't `echo 3 > drop_caches` in production except to confirm a hypothesis;
# it just forces re-reads and masks the real working-set problem.
```

## Decision tree

| Signal | Action |
|---|---|
| low free, high Cached, PSI quiet | normal — no action |
| memory PSI stalling, allocstall rising | true shortage — add RAM / shrink working set / cap cgroups |
| hot set evicted + IO climbs | working set > RAM — more RAM or reduce set |
| kswapd pegged | sustained pressure — see `diagnostics/swap-thrashing.md` |
| SReclaimable huge | slab pressure — see `diagnostics/slab-cache-bloat.md` |

## References

- [Pressure Stall Information (PSI) — kernel.org](https://www.kernel.org/doc/html/latest/accounting/psi.html)
- [Linux ate my RAM — linuxatemyram.com](https://www.linuxatemyram.com/)
- vault: `diagnostics/swap-thrashing.md`, `diagnostics/oom-killed.md`, `diagnostics/slab-cache-bloat.md`, `systems/linux/memory.md`
