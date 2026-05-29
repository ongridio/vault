---
title: Kernel Slab Cache Bloat (Unaccounted Memory)
kind: howto
tags: [memory, slab, kernel, dentry, inode, slabtop, leak]
applies_to: [edge, manager]
---

# Kernel Slab Cache Bloat (Unaccounted Memory)

Use when memory is "used" but no userspace process accounts for it — `ps`
RSS sums far below `MemUsed`, and `Slab` in `/proc/meminfo` is huge. The
kernel's slab allocator caches objects (dentries, inodes, network
buffers, etc.); a workload that opens millions of files or creates huge
numbers of kernel objects bloats slab, and `SUnreclaim` slab can't be
freed under pressure. **Find which slab cache is growing.**

| Symptom | Probable cause |
|---|---|
| `MemUsed` >> sum of process RSS | kernel slab holding the memory |
| `Slab` / `SReclaimable` huge | dentry/inode cache from many file ops |
| `SUnreclaim` huge + OOM risk | non-reclaimable kernel objects (driver/leak) |
| Grows with file-heavy workload | dentry/inode cache (often benign, reclaimable) |

## Step 1 — Quantify slab vs reclaimable

```bash
grep -E 'Slab|SReclaimable|SUnreclaim' /proc/meminfo
# Top slab caches by size
slabtop -o -s c 2>/dev/null | head -20
#   or: sort -k1 -rn /proc/slabinfo (raw)
```

`SReclaimable` large = mostly dentry/inode cache; the kernel will free it
under pressure (usually benign). `SUnreclaim` large = the worry —
non-reclaimable kernel memory that can drive OOM.

## Step 2 — Which cache + who drives it

```bash
slabtop -o -s c | head        # dentry / inode_cache / kmalloc-* / driver caches
# dentry/inode bloat → a process stat()ing or opening huge numbers of files
ls /proc/*/fd 2>/dev/null | wc -l        # gross open-fd sense (also see fd playbook)
```

`dentry` + `inode_cache` dominating = a file-scanning workload (find,
backup, antivirus, a crawler) populated the cache. `kmalloc-*` or a
named driver cache growing = network buffers or a kernel/driver leak.

## Step 3 — Reclaimable bloat (usually fine)

Big reclaimable dentry/inode cache is normal after file-heavy work and is
freed automatically under memory pressure. To confirm it's reclaimable:

```bash
# Hypothesis test ONLY (don't routinely run in prod): force reclaim
sync; echo 2 > /proc/sys/vm/drop_caches    # 2 = dentries+inodes
grep -E 'Slab|SReclaimable' /proc/meminfo   # should drop
```

If it frees, it was benign reclaimable cache — no action needed beyond
maybe `vm.vfs_cache_pressure` tuning if it crowds out the working set.

## Step 4 — Unreclaimable growth = real problem

Monotonically growing `SUnreclaim` that does NOT free under pressure is a
kernel/driver leak or a workload creating unbounded non-reclaimable
objects. Identify the cache (slabtop), map it to a subsystem/driver, and
escalate (kernel/driver update) — userspace tuning won't fix a kernel
leak.

## Decision tree

| Signal | Action |
|---|---|
| `SReclaimable` huge, frees on drop_caches | benign cache; tune `vfs_cache_pressure` if needed |
| dentry/inode from file scanner | expected; cap the scanner / accept reclaimable cache |
| `SUnreclaim` grows, won't free | kernel/driver leak — identify cache, update/escalate |
| network buffer slab huge | check NIC/driver + socket buffers (network playbooks) |

## References

- [slabtop(1) — man7](https://man7.org/linux/man-pages/man1/slabtop.1.html)
- [Slab allocator — kernel.org](https://www.kernel.org/doc/html/latest/mm/slab.html)
- vault: `diagnostics/page-cache-pressure.md`, `diagnostics/fd-exhaustion.md`, `diagnostics/oom-killed.md`, `systems/linux/memory.md`
