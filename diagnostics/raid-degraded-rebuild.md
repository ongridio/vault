---
title: Software RAID Degraded / Rebuild
kind: howto
tags: [raid, mdadm, storage, degraded, rebuild, disk-failure]
applies_to: [edge, manager]
---

# Software RAID Degraded / Rebuild

Use when a Linux `md` array is degraded (a disk dropped), a rebuild is
running (and slowing IO), or you got an mdadm failure alert. A degraded
array still serves data but has **no redundancy** until rebuilt — a
second failure now means data loss. **Identify the failed member, replace
it, and watch the rebuild — and during rebuild, treat the array as
fragile.**

| Symptom | Probable cause |
|---|---|
| `[U_]` / `_U` in /proc/mdstat | a member failed/removed — array degraded |
| Rebuild running, IO slow | resync after replacement consuming bandwidth |
| Repeated member drops | flaky cable/controller, not just the disk |
| `mdadm` failure email/alert | a device marked faulty |

## Step 1 — Array + member state

```bash
cat /proc/mdstat                      # [UU]=healthy, [U_]=degraded, recovery %
mdadm --detail /dev/md0               # State, failed/active devices, which slot
dmesg -T | grep -iE 'md/|raid|I/O error|ata[0-9]|nvme'
```

`[U_]`/`_U` shows a missing member. `mdadm --detail` names the
`faulty`/`removed` device and its slot. dmesg shows whether the disk
threw I/O errors (real failure) or just got kicked.

## Step 2 — Confirm the disk is actually bad

```bash
smartctl -H /dev/sdX; smartctl -A /dev/sdX | grep -iE 'reallocated|pending|crc'
```

A member kicked with clean SMART + no I/O errors may have dropped from a
cable/controller glitch (re-add it). A member with failing SMART / I/O
errors is genuinely dead (replace it). Re-adding a truly bad disk just
fails again.

## Step 3 — Replace + rebuild

```bash
# Mark + remove the failed member (if not already)
mdadm /dev/md0 --fail /dev/sdX --remove /dev/sdX
# Partition the new disk to match, then add it — rebuild starts automatically
mdadm /dev/md0 --add /dev/sdY
watch -n5 'cat /proc/mdstat'          # recovery % + ETA
```

The rebuild reads all surviving members to reconstruct the new one — IO-
and time-intensive. The array is degraded (no redundancy) until it
finishes; a second disk failure during rebuild = data loss, so prioritize.

## Step 4 — Rebuild speed vs service impact

```bash
# Rebuild competes with app IO. Tune the resync speed limits:
sysctl dev.raid.speed_limit_min dev.raid.speed_limit_max
# Raise min to rebuild faster (more IO impact); lower to protect service
sysctl -w dev.raid.speed_limit_min=50000   # KB/s
```

Trade-off: faster rebuild shortens the no-redundancy window but steals IO
from the app. On a critical degraded array, finishing the rebuild fast is
usually worth the temporary IO hit.

## Decision tree

| Signal | Action |
|---|---|
| `[U_]`, SMART/IO errors on member | replace the disk; `--add` new; watch rebuild |
| member kicked, SMART clean | likely cable/controller — re-add; watch for re-drop |
| rebuild slow, array critical | raise `speed_limit_min` to shorten exposure |
| repeated drops, multiple members | controller/backplane/cable — escalate hardware |
| second failure during rebuild | data loss risk — restore from backup |

## References

- [mdadm(8) — man7](https://man7.org/linux/man-pages/man8/mdadm.8.html)
- [Linux RAID wiki — kernel.org](https://raid.wiki.kernel.org/index.php/Linux_Raid)
- vault: `diagnostics/disk-io-saturation.md`, `diagnostics/filesystem-readonly-remount.md`, `systems/storage/block-devices-and-filesystems.md`
