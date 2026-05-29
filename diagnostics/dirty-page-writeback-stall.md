---
title: Dirty-Page Writeback Stalls (Periodic Write Freezes)
kind: howto
tags: [memory, writeback, dirty-pages, vm-dirty-ratio, io, latency]
applies_to: [edge, manager]
---

# Dirty-Page Writeback Stalls (Periodic Write Freezes)

Use when writers freeze periodically — a process writing data stalls for
hundreds of ms to seconds at regular intervals, with IO spiking then
quieting. When dirty (unwritten) pages exceed `vm.dirty_ratio`, the
kernel forces the *writing process itself* to synchronously flush before
it can continue — a write cliff. Big RAM + a slow disk makes this worse
(huge dirty pool dumped onto a slow device at once).

| Symptom | Probable cause |
|---|---|
| Writers freeze at intervals, IO spikes | hit `dirty_ratio` → synchronous writeback |
| `Dirty` in meminfo sawtooths large→small | big dirty pool flushed in bursts |
| Worse on big-RAM + slow-disk hosts | high absolute dirty bytes vs slow drain |
| fsync latency spikes | writeback + journal flush contention |

## Step 1 — Watch the dirty pool

```bash
watch -n1 "grep -E 'Dirty|Writeback' /proc/meminfo"
# Big Dirty that collapses periodically = burst flushing
vmstat 1 5                           # bo (blocks out) spikes during the freeze
```

`Dirty` climbing then collapsing to near-zero in bursts, with `bo`
spiking and the app stalling at those moments = writeback cliffs.

## Step 2 — Current thresholds

```bash
sysctl vm.dirty_ratio vm.dirty_background_ratio
sysctl vm.dirty_bytes vm.dirty_background_bytes   # (0 if ratio-based)
```

Defaults are *percent of RAM*. On a 128 GB host, `dirty_ratio=20` lets
~25 GB of dirty data accumulate before forced sync — dumping that onto a
slow disk freezes writers for a long time.

## Step 3 — Confirm it's the cliff, not device failure

A device that's simply slow/saturated shows steady high `%util` (see
`diagnostics/disk-io-saturation.md`). The writeback-cliff signature is
*bursty*: fine, then a sharp stall when the threshold trips, then fine
again — tied to `Dirty` collapsing.

## Step 4 — Tune for smaller, steadier flushing

```bash
# Cap absolute dirty bytes low so flushing is frequent + small (smoother)
sysctl -w vm.dirty_background_bytes=$((64*1024*1024))   # start bg flush at 64MB
sysctl -w vm.dirty_bytes=$((256*1024*1024))             # hard cliff at 256MB
# (Setting *_bytes overrides the *_ratio percentages)
```

Lower, byte-based thresholds make the kernel write back continuously in
small chunks instead of hoarding then dumping — trading a touch of
throughput for far smoother latency. Persist in sysctl.d.

## Decision tree

| Signal | Action |
|---|---|
| bursty stalls tied to Dirty collapse | set `vm.dirty_bytes`/`dirty_background_bytes` low |
| big RAM + slow disk | byte-based caps (don't let GB of dirty accumulate) |
| steady high %util (not bursty) | device saturated — `diagnostics/disk-io-saturation.md` |
| fsync-bound app | smaller dirty pool + faster journal device |

## References

- [vm sysctl: dirty_ratio / dirty_bytes — kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html)
- [Writeback throttling — LWN](https://lwn.net/Articles/682582/)
- vault: `diagnostics/disk-io-saturation.md`, `diagnostics/page-cache-pressure.md`, `systems/storage/io-scheduling.md`
