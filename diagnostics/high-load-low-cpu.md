---
title: High Load Average But Low CPU
kind: howto
tags: [cpu, load, io-wait, scheduler, d-state]
applies_to: [edge, manager]
---

# High Load Average But Low CPU

Use when `uptime` shows load > number-of-cores but `top` / `mpstat` shows
CPU% well under 100. Load average counts **runnable + uninterruptible**
tasks; CPU% only sees the runnable half. The gap is almost always
**D-state processes blocked on IO** — disk, NFS, lock, network FS.

## Step 1 — Confirm the gap

```bash
uptime                              # load avg
mpstat -P ALL 1 3                   # per-CPU %usr / %sys / %iowait
vmstat 1 5                          # b column = blocked, wa = iowait
```

If `vmstat`'s `b` column (blocked tasks) is consistently > 0 and `wa`
(iowait) is non-trivial, you're in the IO-bound branch. If `b` stays at
0 but load is still high, you've got a runaway scheduler / context
switch storm — different tree, see Step 4.

## Step 2 — Find the D-state processes

```bash
# Snapshot — processes currently in uninterruptible sleep
ps -eo state,pid,user,comm,wchan | awk '$1 ~ /D/'

# wchan tells you WHAT they're waiting on: io_schedule_timeout = block IO,
# wait_on_page_bit = page cache, jbd2_log_wait = ext4 journal, etc.
```

If wchan is empty (kernel doesn't export it), fall back to:

```bash
cat /proc/<pid>/stack          # current kernel call chain
```

Look for `io_submit`, `wait_for_completion_io`, `bio_endio` — all
block-layer.

## Step 3 — Identify the storage culprit

```bash
iostat -xz 1 5                  # %util > 80 = device saturated
                                # await > 100ms = unhealthy latency
iotop -oP                       # top processes by IO (needs root)
pidstat -d 1 5                  # per-process IO/sec
```

For specific filesystems:

```bash
# Slow NFS / FUSE mount
mount | grep -E 'nfs|fuse'      # any of these listed?
nfsstat -c                      # client retransmits
```

If the slow device is a network filesystem, you're not solving a local
IO problem — escalate to the network / storage team.

## Step 4 — Scheduler / context-switch storm (no IO involved)

If Step 1's `b` column was 0 but load is still high:

```bash
vmstat 1 5                      # look at cs (context switches/sec)
pidstat -w 1 5                  # per-process voluntary + nonvoluntary CS
perf top -e context-switches    # who's doing them
```

Causes: thread-pool too large, tight spin locks, futex contention.
`pidstat -w` showing one PID dominating cs is the smoking gun.

## Step 5 — Verify with cgroup

```bash
# If on cgroup v2 (modern systems)
cat /sys/fs/cgroup/<unit>/cpu.stat
# look at throttled_time / throttled_periods
```

Throttled cgroups push load up without spending CPU — appears identical
to D-state from the outside. Common with conservative `CPUQuota=` on
systemd units.

## Decision tree

| Signal | Diagnosis |
|---|---|
| `vmstat b > 0` + `wa > 5%` + D-state procs hitting `io_schedule` | Local disk slow → Step 3 |
| `vmstat b > 0` but `wa < 1%` + D-state on `nfs_*` | Slow NFS → server side |
| `vmstat b == 0` + `cs > 50k/s` | Scheduler thrash → Step 4 |
| `vmstat b == 0` + `cs normal` + `cpu.stat throttled_time` growing | cgroup quota → bump limit |
| All clean but load still high | Likely measurement window — wait 1 min, recompute |

## References

- [Linux Load Averages: Solving the Mystery — Brendan Gregg](https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)
- [iostat output explained — kernel.org](https://www.kernel.org/doc/Documentation/iostats.txt)
- vault: `diagnostics/disk-pressure.md`, `systems/storage/io-scheduling.md`
