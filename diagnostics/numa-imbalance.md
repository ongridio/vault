---
title: NUMA Imbalance / Remote-Memory Latency
kind: howto
tags: [numa, memory, cpu, locality, numactl, latency, performance]
applies_to: [edge, manager]
---

# NUMA Imbalance / Remote-Memory Latency

Use on multi-socket (or large multi-die) servers where a workload is
slower than its CPU/memory usage suggests, with no obvious saturation.
On NUMA hardware, memory attached to one socket is slow to access from
another. A process scheduled on node 1 but with memory on node 0 pays a
remote-access penalty on every miss. **Confirm there's NUMA and that
access is crossing nodes before tuning.**

| Symptom | Probable cause |
|---|---|
| Slow despite idle headroom, multi-socket | cross-node (remote) memory access |
| One NUMA node's memory full, other free | allocation pinned to one node, imbalance |
| High remote-access counters | threads migrated away from their memory |
| Big in-memory DB/cache underperforms | not NUMA-aware; memory split across nodes |

## Step 1 — Is this NUMA hardware, and how is memory split

```bash
numactl --hardware                    # nodes, per-node memory, distances
lscpu | grep -i numa                  # NUMA node → CPU mapping
cat /sys/devices/system/node/node*/meminfo | grep -E 'MemFree|MemUsed' 2>/dev/null
```

One node nearly full while another is free = allocation imbalance. A
single node (distances trivial) means NUMA isn't your issue.

## Step 2 — Measure remote access

```bash
numastat                              # numa_hit vs numa_miss / other_node
numastat -p <pid>                     # per-process node memory distribution
```

A high `numa_miss` / `other_node` ratio means allocations are landing on
a remote node — the latency tax. `numastat -p` shows if a process's
memory is split across nodes instead of local.

## Step 3 — Why it's crossing nodes

- The scheduler migrated threads to another node away from their memory.
- The app allocated all memory on node 0 at startup (first-touch by an
  init thread) then spread work across nodes.
- Interleave policy spreading a latency-sensitive dataset across nodes.

```bash
cat /proc/<pid>/numa_maps | head      # where this process's pages live
```

## Step 4 — Pin for locality

```bash
# Bind a latency-sensitive process to one node (CPU + memory together)
numactl --cpunodebind=0 --membind=0 <cmd>
# Or interleave for bandwidth-bound workloads that span nodes anyway
numactl --interleave=all <cmd>
```

Pin CPU and memory to the same node for latency-sensitive services;
interleave for throughput workloads. Many databases have native NUMA
settings — prefer those over external pinning when available.

## Decision tree

| Signal | Action |
|---|---|
| high `numa_miss`/other_node | pin CPU+memory to same node (`numactl --membind`) |
| one node full, other free | fix first-touch / bind allocation to balance |
| latency-sensitive in-mem app | `--cpunodebind` + `--membind` together |
| bandwidth-bound, spans nodes | `--interleave=all` for even bandwidth |
| single NUMA node | not a NUMA issue — look elsewhere |

## References

- [numactl(8) & NUMA — man7](https://man7.org/linux/man-pages/man8/numactl.8.html)
- [What is NUMA? — kernel.org](https://docs.kernel.org/mm/numa.html)
- vault: `diagnostics/high-latency-p99.md`, `diagnostics/memory-pressure-cn.md`, `systems/linux/memory.md`
