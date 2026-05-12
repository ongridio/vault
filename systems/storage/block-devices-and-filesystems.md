---
title: Block Devices and Filesystems
tags: [storage, block, filesystem, ext4, xfs]
---

# Block Devices and Filesystems

The Linux storage stack runs **block device → IO scheduler → filesystem
→ page cache → VFS**. Most operator problems live at the filesystem or
the IO-scheduler boundary. This doc names the layers so you can describe
a problem precisely.

## The stack

```
application → VFS → filesystem (ext4 / xfs) → page cache
            ↓
       block layer → IO scheduler (mq-deadline / bfq / none)
            ↓
       device driver (nvme, scsi, virtio-blk)
            ↓
       physical media
```

`/sys/block/<dev>/queue/scheduler` shows the active scheduler for each
device. NVMe devices use `none` by default (the device queue depth + the
device's own scheduling are richer than anything the kernel can add).
SATA SSDs are usually `mq-deadline`. Rotational disks may still benefit
from `bfq` if the workload mixes interactive and batch IO.

## Filesystem choice

| FS    | When to pick it | Watch out for |
|-------|----------------|---------------|
| ext4  | Default; well-understood; small files OK | inode count is fixed at mkfs |
| xfs   | Big files, many parallel writers, large volumes | xfs_repair on corruption is slower than e2fsck |
| btrfs | Snapshots / subvolumes are core to the workflow | metadata cost; balance / scrub overhead |
| zfs   | Multi-disk arrays, checksumming, compression | not in the upstream kernel; ARC RAM appetite |

Ongrid edge nodes default to **ext4** for the OS volume and **xfs** for
the data volume (Prometheus TSDB / Loki chunks / Tempo blocks all do
better on xfs's allocator under heavy parallel writes).

## inodes — the silent fail

`df -i` is the question most operators skip. An inode-exhausted ext4
volume reports plenty of free blocks but refuses new files. Common
producers:

- Container layered filesystems (`/var/lib/docker/overlay2/*/diff/`)
- Mail spools / Maildir-style
- npm / pip caches under per-user directories

ext4's `mke2fs -i <ratio>` sets the bytes-per-inode at format time. Once
formatted, you cannot add inodes — only reformat. xfs allocates inodes
dynamically so it doesn't have this failure mode (it does have a
metadata fragmentation failure mode instead).

## Page cache mechanics

Reads and writes go through the page cache by default. Implications:

- **Writes are async.** `write(2)` returns when the kernel has the page,
  not when the device has it. `fsync(2)` / `O_DIRECT` change that.
- **Hot working set lives in RAM.** A "slow disk" alert that disappears
  after `echo 3 > /proc/sys/vm/drop_caches` was actually a cold-cache
  benchmark, not a hardware problem.
- **Page cache competes with anonymous memory.** Under memory pressure
  the kernel reclaims clean cache pages first; that's why cache shrinks
  *before* swap kicks in.

## Mount options that matter

```
ext4 defaults,noatime,nodiratime,discard,data=ordered
xfs  defaults,noatime,nodiratime,inode64,largeio
```

- `noatime` removes the per-read inode update — huge win on read-heavy
  workloads
- `discard` issues TRIM on every delete — modern SSDs handle this fine;
  cheap SSDs may stutter. For those use a weekly `fstrim.timer` instead
- `inode64` on xfs lets inode numbers exceed 32 bits — required for
  volumes > a few TB

## Telltale signs

| Symptom | Likely layer |
|---------|--------------|
| `df -h` says full, `du` says less | deleted-but-held file (`lsof | grep deleted`) |
| Writes slow, reads fine | journal / wal congestion (xfs log size, ext4 commit interval) |
| Random latency spikes on SSD | TRIM / GC on a consumer-grade drive |
| All IO slow under load | scheduler / queue depth / `nr_requests` |
| Specific file slow | fragmentation (`filefrag`); on xfs check `xfs_db` extents |

## See also

- `systems/storage/io-scheduling.md` — scheduler and queue depth
- `diagnostics/disk-pressure.md` — operator walkthrough for high usage
- `alerts/host-metrics.md#disk_high` — alert handler
