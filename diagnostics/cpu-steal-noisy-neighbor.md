---
title: CPU Steal Time / Noisy Neighbor (VMs)
kind: howto
tags: [cpu, steal, virtualization, hypervisor, noisy-neighbor, vm]
applies_to: [edge, manager]
---

# CPU Steal Time / Noisy Neighbor (VMs)

Use when a VM/cloud instance is slow but its *own* CPU usage looks
moderate — and `%steal` (`st`) in `top`/`vmstat` is non-trivial. Steal
is time the hypervisor wanted to run your vCPU but couldn't, because a
co-tenant (or your own throttle) held the physical core. **You can't fix
steal from inside the guest — you confirm it, then move or resize.**

| Symptom | Probable cause |
|---|---|
| `%steal` consistently > a few % | overcommitted host / noisy neighbor |
| Steal spikes at specific times | a co-tenant's batch job on the same host |
| Steal + burstable instance, credits gone | cloud CPU-credit throttling (you're the throttle) |
| No steal, still slow | not steal — look elsewhere (IO, throttle, app) |

## Step 1 — Quantify steal

```bash
vmstat 1 5                            # 'st' column = % stolen
mpstat -P ALL 1 5                     # %steal per vCPU
top                                   # 'st' in the CPU line
```

Sustained `%steal` above ~5–10% means a meaningful slice of your CPU is
being taken. Compare across vCPUs — uniform steal = host overcommit;
uneven = scheduling/affinity on the host side.

## Step 2 — Burstable instance? Check credits (cloud)

On burstable instance types (AWS T-series, GCP e2 / shared-core), once
CPU credits run out the platform throttles you — and that throttling can
*appear* as steal. Check the provider's CPU-credit-balance metric. If
credits are exhausted, you're not a noisy-neighbor victim; you've
outgrown the instance class.

## Step 3 — Confirm it's not in-guest throttling

```bash
# A cgroup CPU quota inside the guest looks different from steal —
# rule it out so you don't blame the hypervisor for your own limit
cat /sys/fs/cgroup/<svc>/cpu.stat | grep throttled   # see cpu-throttling playbook
```

Steal = hypervisor took the core. cgroup throttle = your own quota. They
need opposite fixes; don't conflate them.

## Step 4 — Remediate (all outside the guest)

- **Move the instance** to a less-contended host (stop/start often
  re-places it on cloud).
- **Resize** to a dedicated / non-burstable / larger class.
- **Dedicated tenancy / CPU pinning** for latency-critical workloads.
- On your own hypervisor (KVM/VMware/Proxmox): reduce vCPU overcommit
  ratio, pin critical guests, or rebalance noisy guests off the host.

## Decision tree

| Signal | Action |
|---|---|
| Uniform high `%steal` | host overcommitted — move/resize the instance |
| Burstable, credits depleted | upgrade instance class; you're throttled, not stolen-from |
| Steal at specific times | co-tenant batch job — move to dedicated host |
| Self-managed hypervisor | lower vCPU overcommit, pin/rebalance guests |
| `%steal` ~0 but slow | not steal — see throttling / IO / app playbooks |

## References

- [Brendan Gregg — CPU steal & utilization](https://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html)
- [KVM CPU overcommit — linux-kvm.org](https://www.linux-kvm.org/page/Documents)
- vault: `diagnostics/cpu-throttling.md`, `diagnostics/high-load-low-cpu.md`, `systems/scheduling/cfs-and-cgroups.md`
