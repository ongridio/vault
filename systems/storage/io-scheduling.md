---
title: Linux IO Scheduling
tags: [storage, io, scheduler, blk-mq]
---

# Linux IO Scheduling

Block IO scheduling sits between the filesystem and the device. Pick the
wrong scheduler and a perfectly healthy SSD looks like a degraded one,
or vice-versa.

## The schedulers

Modern Linux uses **blk-mq** (multi-queue block layer). Available
schedulers:

| Scheduler | Best for | Trade-off |
|-----------|---------|-----------|
| `none`    | NVMe; high-IOPS SSDs | No fairness — device's own queue handles ordering |
| `mq-deadline` | SATA SSDs, mixed workloads | Bounds latency by deadline; modest CPU cost |
| `bfq`     | Rotational disks; desktop; mixed interactive+batch | Strong fairness; higher CPU cost; can underutilize fast SSDs |
| `kyber`   | Fast SSDs with latency targets | Tunes itself; rarely necessary in practice |

Check / set:

```bash
cat /sys/block/<dev>/queue/scheduler
echo mq-deadline > /sys/block/<dev>/queue/scheduler
```

Make it persistent via udev:

```
ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/scheduler}="mq-deadline"
```

## Queue depth knobs

`nr_requests` (per-queue depth) controls how many IOs the kernel will
issue before applying back-pressure to the filesystem. Defaults are
conservative for general-purpose workloads but undersized for high-IOPS
TSDBs.

```bash
cat /sys/block/<dev>/queue/nr_requests       # current
echo 1024 > /sys/block/<dev>/queue/nr_requests
```

`read_ahead_kb` controls speculative reads. Sequential workloads
(Prometheus blocks, Loki chunks) benefit from raising it; random
workloads (MySQL InnoDB) want it small.

```bash
echo 4096 > /sys/block/<dev>/queue/read_ahead_kb  # heavy sequential
echo 128  > /sys/block/<dev>/queue/read_ahead_kb  # random
```

## Reading `iostat -xz 1`

```
Device  r/s   w/s   rkB/s  wkB/s  await  svctm  %util
nvme0n1 1200  3400  9600   27200  0.42   0.15   72.30
```

- **r/s + w/s** — IOPS
- **kB/s** — throughput
- **await** — average time from submit to completion, ms (includes queue
  time)
- **svctm** — average device service time, ms (deprecated; iostat still
  prints it but newer versions warn)
- **%util** — fraction of wall time the device had at least one IO in
  flight

Rules of thumb:

- `%util` near 100% with low `await` → device saturated but coping
- `%util` near 100% with high `await` → genuinely overloaded; latency
  will degrade further as load rises
- `%util` low with high `await` → device fine, something else queueing
  (often filesystem journal, or another consumer on the same channel)

## CFQ is gone

The single-queue era's CFQ scheduler is removed in modern kernels.
Anyone telling you to "set CFQ for fairness on SSDs" is reading old
docs — use `bfq` if you actually need that behavior.

## When to tune, when not to

Default schedulers are correct **for most workloads on most hardware**.
Reach for tuning only when you can measure the win:

1. Baseline `fio` workload that matches production access pattern
2. Change one knob, re-run, compare
3. If the difference is < ~10%, leave the default — the next kernel
   upgrade might invert it
