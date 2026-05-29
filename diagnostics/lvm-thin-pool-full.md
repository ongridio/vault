---
title: LVM Thin Pool Full (Data or Metadata)
kind: howto
tags: [lvm, thin-pool, storage, overprovision, snapshot, metadata]
applies_to: [edge, manager]
---

# LVM Thin Pool Full (Data or Metadata)

Use when thin-provisioned LVM volumes start erroring on write, or
filesystems on thin LVs go read-only, even though the *filesystem* shows
free space. Thin pools overprovision: the sum of thin volumes can exceed
the pool's real capacity, and when the pool's **data** or **metadata**
fills, writes fail hard. **Check the pool, not just the LV/filesystem.**

| Symptom | Probable cause |
|---|---|
| Writes fail, FS shows free space | underlying thin pool data full |
| `Thin pool ... is now X% full` in logs | pool approaching/at capacity |
| Metadata-related errors, pool not data-full | thin pool *metadata* exhausted |
| Sudden after snapshots | snapshots consuming pool blocks |

## Step 1 — Pool data + metadata usage

```bash
lvs -a -o +data_percent,metadata_percent,lv_size,pool_lv
#   Data% AND Meta% both matter — either at 100% breaks writes
dmsetup status | grep thin           # low-level pool status
```

`Data%` at/near 100 = data exhausted. `Meta%` at 100 (even with data
free) also breaks the pool — metadata exhaustion is the sneaky one.

## Step 2 — What consumed it

```bash
lvs -a -o +data_percent,origin       # snapshots/thin LVs and their usage
# Overprovision check: sum of thin LV sizes vs pool size
vgs -o +vg_free
```

Common causes: more data written than the pool physically holds
(overprovisioned and you actually used it), or snapshots accumulating
changed blocks. Deleted files don't free pool space unless `discard`/fstrim
runs (see Step 4).

## Step 3 — Recover (carefully — full pools are fragile)

```bash
# Extend the pool data (if the VG has free PEs / you can add a PV)
lvextend -L +20G <vg>/<thinpool>
# Extend metadata if Meta% is the problem
lvextend --poolmetadatasize +1G <vg>/<thinpool>
```

If the VG has no free space, add a physical volume first (`pvcreate` +
`vgextend`). A 100%-full thin pool can wedge — extend data/metadata
before trying to mount/write more. Activating a full pool may require the
extend first.

## Step 4 — Reclaim + prevent

```bash
fstrim -av                            # return freed FS blocks to the pool (needs discard)
# Ensure discard passes through: filesystem mounted with discard or periodic fstrim,
# and the thin pool created with --discards passdown
```

Prevent recurrence: enable `fstrim.timer`, set thin-pool autoextend
thresholds (`thin_pool_autoextend_threshold`), monitor `Data%`/`Meta%`
as alerts, and don't overprovision beyond what you can extend.

## Decision tree

| Signal | Action |
|---|---|
| Data% 100% | extend pool data (lvextend); add PV if VG full |
| Meta% 100% | extend pool metadata (`--poolmetadatasize`) |
| space not reclaimed after deletes | run `fstrim`; enable discard passdown |
| snapshots eating pool | prune old snapshots; cap snapshot policy |
| recurring | set autoextend threshold + monitor Data%/Meta% |

## References

- [lvmthin(7) — man7](https://man7.org/linux/man-pages/man7/lvmthin.7.html)
- [LVM(8) — man7](https://man7.org/linux/man-pages/man8/lvm.8.html)
- vault: `diagnostics/disk-full-cn.md`, `diagnostics/filesystem-readonly-remount.md`, `systems/storage/block-devices-and-filesystems.md`
