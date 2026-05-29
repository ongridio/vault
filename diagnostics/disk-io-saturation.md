---
title: Disk I/O Saturation & High iowait
kind: howto
tags: [disk, io, iowait, iostat, latency, storage, saturation]
applies_to: [edge, manager]
---

# Disk I/O Saturation & High iowait

Use when the box feels slow, load is high but CPU is mostly idle, and
`vmstat`'s `wa` (iowait) is elevated. iowait means CPUs are idle *only
because everything is blocked waiting on storage*. **Separate "the
device is saturated" (high util + queue) from "the device is slow per
op" (high await, low util) from "one process is hammering it".**

| Symptom | Probable cause |
|---|---|
| `%util` ~100%, queue building | device throughput saturated |
| high `await`, modest `%util` | per-op latency (failing disk, noisy neighbor, network storage) |
| iowait high, one PID dominates | a single heavy writer/reader |
| bursty stalls every ~5–30s | writeback flush storm (dirty pages) |

## Step 1 — Per-device throughput, util, and latency

```bash
iostat -xz 1 5                       # await, r/s w/s, rkB/s wkB/s, %util, aqu-sz
# Key columns:
#   %util ~100  -> device busy ceiling
#   await       -> avg ms per IO (rising = latency problem)
#   aqu-sz      -> avg queue depth (deep + slow = backed up)
```

A device pinned at `%util` 100 with a growing `aqu-sz` is saturated —
you're asking for more IOPS/bandwidth than it delivers. High `await`
with low `%util` points at a sick disk or remote-storage latency.

## Step 2 — Who is doing the IO

```bash
iotop -oPa 2>/dev/null                # accumulated IO per process (needs root)
pidstat -d 1 5                        # kB_rd/s kB_wr/s per process
```

If one PID owns the IO, that's your lever (throttle it, fix its access
pattern, move it). If it's spread thin, the device is just undersized
for aggregate load.

## Step 3 — Reads vs writes vs writeback

```bash
cat /proc/meminfo | grep -E 'Dirty|Writeback'   # dirty pages pending flush
sysctl vm.dirty_ratio vm.dirty_background_ratio  # when the kernel forces flush
```

Periodic stalls that line up with `Dirty` collapsing = writeback storms:
the kernel hit `dirty_ratio` and forced synchronous flushing, freezing
writers. Lower the ratios / `dirty_bytes` for smoother, smaller flushes.

## Step 4 — Is the disk itself failing?

```bash
dmesg -T | grep -iE 'I/O error|ata|nvme|medium error|reset'
smartctl -A /dev/sdX 2>/dev/null | grep -iE 'reallocated|pending|crc|wear'
```

I/O errors / ATA resets in dmesg, or rising SMART reallocated/pending
sectors, mean the drive is dying — migrate data and replace, don't tune.

## Decision tree

| Signal | Action |
|---|---|
| `%util`~100, queue deep | device saturated — throttle writer, add IOPS, spread load |
| high `await`, low `%util` | slow device / remote-storage latency — check backend, SMART |
| one PID dominates IO | throttle/fix that process (ionice, cgroup io.max) |
| stalls track `Dirty` collapse | tune `vm.dirty_*` for smaller, steadier writeback |
| dmesg I/O errors / SMART bad | disk failing — replace; this is not a tuning problem |

## References

- [Brendan Gregg — USE Method (saturation)](https://www.brendangregg.com/usemethod.html)
- [iostat(1) — man7](https://man7.org/linux/man-pages/man1/iostat.1.html)
- [LVM(8) — man7](https://man7.org/linux/man-pages/man8/lvm.8.html)
- vault: `diagnostics/high-load-low-cpu.md`, `diagnostics/disk-pressure.md`, `systems/storage/io-scheduling.md`, `systems/storage/block-devices-and-filesystems.md`
