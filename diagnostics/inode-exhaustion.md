---
title: Inode Exhaustion (Disk Has Space But Writes Fail)
kind: howto
tags: [disk, inode, filesystem, ENOSPC, df, storage]
applies_to: [edge, manager]
---

# Inode Exhaustion (Disk Has Space But Writes Fail)

Use when writes fail with `No space left on device` (ENOSPC) but `df -h`
shows plenty of free space. A filesystem has a fixed pool of *inodes*
(one per file/dir); millions of tiny files exhaust the inode table long
before the byte capacity. **Always check `df -i`, not just `df -h`.**

| Symptom | Probable cause |
|---|---|
| ENOSPC, `df -h` shows free space | inode table full (`df -i` at 100%) |
| Concentrated in one dir tree | a runaway file producer (cache, sessions, mail, tmp) |
| Slowly over weeks | unbounded growth — no rotation/cleanup |
| Lots of zero-byte / tiny files | the classic inode-burner pattern |

## Step 1 — Confirm it's inodes, not bytes

```bash
df -h <mount>            # bytes — looks fine
df -i <mount>            # inodes — IUse% at 100% is the smoking gun
```

If `IUse%` is 100 while `Use%` is low, it's inodes. (Note: some
filesystems like XFS/Btrfs allocate inodes dynamically and rarely hit
this — it's mostly ext4 with a low `bytes-per-inode` ratio.)

## Step 2 — Find where the files are

```bash
# Count files per immediate subdir under the full mount (find the hot tree)
for d in /path/*/; do printf "%8d %s\n" "$(find "$d" -xdev 2>/dev/null | wc -l)" "$d"; done | sort -rn | head
# Or quickly: directories with the most entries
find /path -xdev -type d -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head
```

`-xdev` keeps `find` from crossing into other mounts. The top tree is
your producer — usually session files, a cache dir, mail spool, or log
fragments.

## Step 3 — Clean up safely + stop the source

```bash
# Delete by age (don't rm -rf the whole tree blindly)
find /path/cache -xdev -type f -mtime +7 -delete
# A huge dir makes rm slow; for an entire disposable tree, rsync-empty is faster:
#   mkdir /tmp/empty && rsync -a --delete /tmp/empty/ /path/junk/
```

Then fix the source: add rotation/TTL, cap the cache, or change the app
to not spray tiny files.

## Step 4 — If the FS is structurally under-provisioned

ext4's inode count is fixed at `mkfs` time. If a workload legitimately
needs millions of small files, the filesystem was made with too few
inodes — the durable fix is reformatting with `mkfs.ext4 -i <bytes-per-inode>`
(smaller ratio = more inodes) or moving to XFS/Btrfs which allocate
inodes dynamically. Plan migration; you can't add inodes in place.

## Decision tree

| Signal | Action |
|---|---|
| `df -i` 100%, `df -h` free | inode exhaustion — find + prune the file producer |
| One tree owns the inodes | add rotation/TTL there; cap it |
| Recurs after cleanup | structural — re-mkfs with more inodes or move to XFS |
| `df -h` also full | it's bytes, not inodes — see `diagnostics/disk-full-cn.md` |

## References

- [df(1) — man7](https://man7.org/linux/man-pages/man1/df.1.html)
- [ext4 Documentation — kernel.org](https://docs.kernel.org/filesystems/ext4/index.html)
- [XFS Documentation — kernel.org](https://docs.kernel.org/filesystems/xfs/index.html)
- vault: `diagnostics/disk-full-cn.md`, `diagnostics/disk-pressure.md`, `systems/storage/block-devices-and-filesystems.md`
