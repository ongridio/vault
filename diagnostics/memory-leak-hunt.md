---
title: Memory Leak — From RSS Growth to Pmap
kind: howto
tags: [memory, leak, rss, pmap, valgrind, jemalloc]
applies_to: [edge, manager]
---

# Memory Leak — From RSS Growth to Pmap

Use when a process's RSS / VSZ trends upward and never plateaus, OR
when you're already in OOM aftermath asking "who grew unboundedly".
Two roads diverge early: **userspace heap** vs **kernel slabs**. Test
which one first.

## Step 0 — Confirm there IS a leak

Sustained growth over time, not a one-shot spike:

```bash
# Snapshot RSS for the suspect every minute for 10 min
for i in $(seq 10); do
  ps -o rss=,vsz= -p <pid>
  sleep 60
done
```

Constant or oscillating RSS = not a leak. Monotonic up = leak (or
cache warming, see Step 1).

## Step 1 — Rule out caches before chasing application code

```bash
free -h                           # buff/cache vs used
cat /proc/meminfo | grep -E 'Slab|SReclaimable|SUnreclaim|Cached|Buffers'
slabtop -o | head -20             # kernel object allocations
```

If the growing memory is in:
- `Cached` / `Buffers` — page cache, will release under pressure. Not a leak.
- `SReclaimable` — dentry / inode cache, releases under pressure. Not a leak unless growing AND unreclaimable.
- `SUnreclaim` — **real** kernel leak (driver / module / netfilter table). Go to Step 5.

## Step 2 — Where in the process is memory growing

```bash
# Two snapshots of /proc/<pid>/status, 1 min apart
cat /proc/<pid>/status | grep -E '^Vm|^Rss'

# Region-level: pmap with -X for kernel detail
pmap -x <pid> > /tmp/pmap.before
sleep 60
pmap -x <pid> > /tmp/pmap.after
diff /tmp/pmap.before /tmp/pmap.after | head -50
```

Look for `[heap]` size growing → application allocator leak. `[anon]`
mappings growing → mmap'd buffers (often DBs, JIT runtimes). Specific
shared library `.so` growing → that library's allocator.

## Step 3 — Allocator-level visibility (glibc / jemalloc / tcmalloc)

For glibc malloc:

```bash
# Force the process to dump malloc stats to its stderr
gdb --pid=<pid> --batch \
  -ex 'call (void) malloc_stats()' \
  -ex 'detach' -ex 'quit'
```

For jemalloc-linked binaries:

```bash
# Trigger jemalloc heap profile dump via signal or runtime control
# (process must have been started with MALLOC_CONF="prof:true,prof_active:true,...")
jeprof --show_bytes --text /path/to/binary jeprof.out.<pid>.* | head -30
```

For Go programs:

```bash
# If pprof endpoint is exposed
go tool pprof -top http://<host>:<port>/debug/pprof/heap
# Look at -alloc_space vs -inuse_space — the latter is the leak
```

## Step 4 — Last-resort: per-allocation trace

```bash
# eBPF-based: counts allocations by stack
sudo /usr/share/bcc/tools/memleak -p <pid>          # bcc
# Or: profile.py for sampling
```

`memleak` shows top allocation stacks that haven't been freed within
its observation window. Best signal but expensive — only attach for 1-2
min on a hot process.

## Step 5 — Kernel leak (SUnreclaim growth)

```bash
slabtop -o -s c | head -20         # top object types by total size
# Look for unusual growth in: kmalloc-*, dentry, inode_cache, vm_area_struct,
# nf_conntrack, sock_inode_cache
```

| Growing slab | Probable cause |
|---|---|
| `nf_conntrack` | conntrack table not aging out — see vault `diagnostics/conntrack-table-full.md` |
| `dentry` / `inode_cache` | `find /` ran wild or rapid open/close pattern |
| `kmalloc-*` (256/512/1024) | Module / driver bug — check `lsmod` for recently loaded |
| `sock_inode_cache` | Socket leak in userspace not closing FDs |

```bash
# Trigger reclaim — if memory frees, it was cache; if not, leak
sync && echo 3 > /proc/sys/vm/drop_caches
free -h          # compare before/after
```

## Step 6 — If it's a container

```bash
# Per-container memory and cgroup-level pressure
docker stats --no-stream
cat /sys/fs/cgroup/<unit>/memory.current
cat /sys/fs/cgroup/<unit>/memory.peak
cat /sys/fs/cgroup/<unit>/memory.pressure
```

cgroup `memory.peak` tells you the high-water mark since start.
`memory.pressure` (PSI) >0 means the cgroup is actively under memory
pressure — limit too small even if no OOM yet.

## Decision tree

| Signal | Next step |
|---|---|
| RSS oscillates within range | Not a leak, stop |
| RSS monotonic up, `free` shows `cached` growing too | Page cache only — fine |
| RSS up + pmap `[heap]` growing | userspace allocator → Step 3 |
| RSS up + specific `[anon]` mapping growing | mmap leak (DB / JIT) — check that library |
| RSS up + `SUnreclaim` up | kernel slab leak → Step 5 |
| Memory grows then dies on OOM | See `diagnostics/disk-pressure.md` for OOM aftermath, here for the cause |

## References

- [Linux memory accounting — kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory)
- [Brendan Gregg — Linux Memory Performance Tools](https://www.brendangregg.com/Perf/linux_observability_tools.html)
- [jemalloc heap profiling guide](https://github.com/jemalloc/jemalloc/wiki/Use-Case%3A-Heap-Profiling)
- vault: `systems/linux/memory.md`, `diagnostics/high-load-low-cpu.md`
