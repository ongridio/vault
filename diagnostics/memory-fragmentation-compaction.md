---
title: Memory Fragmentation / High-Order Allocation Failures
kind: howto
tags: [memory, fragmentation, compaction, high-order, buddy, allocation]
applies_to: [edge, manager]
---

# Memory Fragmentation / High-Order Allocation Failures

Use when allocations fail or stall even though there's plenty of *total*
free memory — `page allocation failure: order:N` in dmesg, or latency
when the kernel needs a contiguous multi-page block. Free memory can be
scattered into single pages (fragmented) so no contiguous high-order
block exists. The kernel then compacts (stall) or fails the allocation.

| Symptom | Probable cause |
|---|---|
| `page allocation failure order:N` in dmesg | no contiguous high-order block free |
| Stalls when net/driver needs big buffers | fragmentation + sync compaction |
| Free memory high but alloc fails | fragmented into low-order pages |
| Worsens with uptime under churn | long-running fragmentation accumulation |

## Step 1 — Inspect the buddy allocator + dmesg

```bash
cat /proc/buddyinfo               # free blocks per order, per zone
dmesg -T | grep -iE 'page allocation failure|order:[2-9]'
grep -E 'compact_stall|compact_fail' /proc/vmstat
```

`buddyinfo` with lots of `order-0`/`order-1` free but ~zero higher orders
= fragmented. `order:N` failures in dmesg (N≥3) confirm a high-order
allocation couldn't be satisfied. Rising `compact_stall` = the kernel is
paying to compact.

## Step 2 — Who needs high-order allocations

High-order allocs come from: network drivers (jumbo/aggregated buffers),
some filesystems, hugepage formation, certain drivers. The dmesg stack on
the failure names the caller. If it's a NIC driver, large `MTU`/jumbo or
GRO buffers may be the demand.

## Step 3 — Fragmentation index

```bash
cat /sys/kernel/debug/extfrag/extfrag_index 2>/dev/null   # if debugfs mounted
# Trigger a compaction pass (per node)
echo 1 > /proc/sys/vm/compact_memory
cat /proc/buddyinfo                # did higher orders reappear?
```

If manual compaction restores higher-order blocks, fragmentation was the
issue — but it'll re-fragment under the same workload.

## Step 4 — Mitigations

- **Lower the demand**: if a driver/MTU forces big contiguous buffers,
  reducing jumbo/MTU or enabling scatter-gather avoids high-order needs.
- **Proactive compaction**: `vm.compaction_proactiveness` (newer kernels)
  keeps fragmentation lower in the background.
- **Reserve at boot**: for predictable high-order needs (hugepages),
  reserve them at boot (`hugepages=` / CMA) when memory is unfragmented.
- A reboot defragments — but that's a band-aid; address the demand.

## Decision tree

| Signal | Action |
|---|---|
| `order:N` failures, buddyinfo fragmented | reduce high-order demand (MTU/jumbo/driver) |
| compaction restores, re-fragments | enable proactive compaction; reserve at boot |
| hugepage formation failing | reserve hugepages at boot, not at runtime |
| failure stack = specific driver | update/configure that driver; scatter-gather |

## References

- [/proc/buddyinfo & fragmentation — kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html)
- [Memory compaction — LWN](https://lwn.net/Articles/368869/)
- vault: `diagnostics/transparent-hugepage-stalls.md`, `diagnostics/page-cache-pressure.md`, `systems/linux/memory.md`
