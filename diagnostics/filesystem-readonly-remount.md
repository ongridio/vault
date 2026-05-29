---
title: Filesystem Remounted Read-Only (errors=remount-ro)
kind: howto
tags: [filesystem, readonly, ext4, xfs, io-error, corruption, remount]
applies_to: [edge, manager]
---

# Filesystem Remounted Read-Only (errors=remount-ro)

Use when writes suddenly fail with `Read-only file system` (EROFS) though
the mount was read-write, and apps/databases start erroring. ext4's
default `errors=remount-ro` flips a filesystem read-only the moment the
kernel detects a metadata error or the underlying device throws an I/O
error — a **protective** response. **This almost always means hardware /
device trouble, not a config mistake. Don't just remount rw — find why.**

| Symptom | Probable cause |
|---|---|
| Sudden EROFS, was rw | kernel hit FS/IO error → `remount-ro` triggered |
| `I/O error`, `ata`/`nvme` reset in dmesg | failing disk / cable / controller |
| EXT4-fs error / metadata corruption | filesystem inconsistency |
| Only after a power loss / crash | unclean shutdown left FS inconsistent |

## Step 1 — Confirm + read why

```bash
mount | grep ' ro,'                          # which mounts went read-only
dmesg -T | grep -iE 'EXT4-fs error|XFS|I/O error|remount|read-only|ata[0-9]|nvme|medium error'
```

The dmesg lines before the remount-ro are the cause: an `I/O error` /
`ata reset` / `nvme` error (device) or an `EXT4-fs error` (metadata
corruption). That distinction drives everything.

## Step 2 — Device health (the common root)

```bash
smartctl -H /dev/sdX; smartctl -A /dev/sdX | grep -iE 'reallocated|pending|crc|wear'
dmesg -T | grep -iE 'ata[0-9]|reset|link is down|timeout'
```

SMART failing / rising reallocated-pending sectors / ATA resets = the
drive (or cable/controller) is dying. **Get backups off it before
anything else** — a forced rw remount on a failing disk risks more
damage.

## Step 3 — If it's filesystem corruption (device healthy)

```bash
# Unmount (stop the apps using it first), then fsck — NEVER fsck a mounted rw FS
umount /mnt/data
fsck -n /dev/sdX          # dry run first (report only)
fsck -y /dev/sdX          # repair (after backup / when sure)
# XFS:
xfs_repair -n /dev/sdX    # dry run; drop -n to repair
```

fsck/xfs_repair on a **mounted** writable filesystem can destroy it —
unmount first (or boot to rescue for the root FS).

## Step 4 — Recover + prevent

```bash
# After repair + device confirmed OK, remount rw
mount -o remount,rw /mnt/data
```

Prevention: monitor SMART + dmesg I/O errors as alerts so you catch the
dying disk before it forces read-only mid-service. For root FS this often
needs a maintenance reboot / rescue boot to fsck cleanly.

## Decision tree

| Signal | Action |
|---|---|
| dmesg I/O error / ATA-NVMe reset | failing device — back up now, replace disk |
| SMART failing / sectors rising | disk dying — migrate data, replace |
| EXT4/XFS metadata error, device OK | unmount + fsck/xfs_repair (after backup) |
| after power loss | unclean shutdown — fsck; add UPS / barriers |
| recurs after remount rw | underlying device still bad — don't keep forcing rw |

## References

- [ext4 errors= mount option — kernel.org](https://docs.kernel.org/admin-guide/ext4.html)
- [fsck(8) / e2fsck(8) — man7](https://man7.org/linux/man-pages/man8/fsck.8.html)
- vault: `diagnostics/disk-io-saturation.md`, `diagnostics/disk-full-cn.md`, `systems/storage/block-devices-and-filesystems.md`
