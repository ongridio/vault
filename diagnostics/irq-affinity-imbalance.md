---
title: IRQ Affinity Imbalance (One CPU Pinned by Interrupts)
kind: howto
tags: [irq, interrupts, affinity, smp, nic, irqbalance, cpu]
applies_to: [edge, manager]
---

# IRQ Affinity Imbalance (One CPU Pinned by Interrupts)

Use when one CPU sits at 100% (often in `%irq`/`%soft`) while the rest
idle, and throughput is capped by that single core. Hardware interrupts
(NIC queues, NVMe, etc.) are all landing on one CPU instead of spread.
This caps packet/IO processing at single-core speed regardless of how
many cores you have. **Confirm the interrupts are concentrated, then
spread them.**

| Symptom | Probable cause |
|---|---|
| One CPU 100% `%irq`/`%soft`, others idle | all device IRQs pinned to one CPU |
| NIC/NVMe throughput capped below HW | single-core interrupt bottleneck |
| `irqbalance` not running | no automatic spreading |
| Multi-queue device, single CPU used | queue→CPU affinity not configured |

## Step 1 — Which CPU gets the interrupts

```bash
# Per-CPU interrupt counts — is one column dominating?
cat /proc/interrupts | awk 'NR==1 || /eth|ens|enp|nvme|mlx|virtio/'
# Watch live concentration
mpstat -P ALL 1 3                     # %irq + %soft per CPU
```

A device's IRQ rows with all counts on CPU0 = pinned. The hot CPU in
`mpstat` with high `%irq` confirms it.

## Step 2 — Affinity + irqbalance

```bash
# Where is a given IRQ allowed to run? (bitmask)
cat /proc/irq/<N>/smp_affinity_list
systemctl is-active irqbalance        # is the balancer even running?
```

`irqbalance` inactive on a multi-core box is a common cause — nothing
spreads the IRQs. (Note: for some high-performance NIC setups operators
*disable* irqbalance and pin manually — then manual affinity must be
correct.)

## Step 3 — Multi-queue: use the queues

```bash
ethtool -l <iface>                    # combined channels (RX/TX queues)
ethtool -L <iface> combined N         # enable more queues so IRQs spread to N CPUs
```

A multi-queue NIC with only one active queue funnels everything to one
IRQ/CPU. Enabling more queues (RSS) gives each queue its own IRQ on a
different CPU.

## Step 4 — Pin or balance

```bash
# Spread: start irqbalance
systemctl enable --now irqbalance
# Or pin a specific IRQ to a CPU set (manual tuning)
echo <cpu-mask> > /proc/irq/<N>/smp_affinity
```

For latency-critical NICs, manual affinity (one queue per dedicated CPU,
aligned with RPS/RFS) outperforms irqbalance — but only if done
correctly; otherwise let irqbalance handle it.

## Decision tree

| Signal | Action |
|---|---|
| one CPU `%irq` 100%, irqbalance off | start irqbalance to spread |
| multi-queue NIC, one queue | enable more channels (`ethtool -L combined`) |
| manual pinning, still on one CPU | fix `smp_affinity` masks per IRQ/queue |
| %soft (not %irq) dominant | softirq, not hard IRQ — `softirq-ksoftirqd-high.md` |

## References

- [SMP IRQ affinity — kernel.org](https://www.kernel.org/doc/html/latest/core-api/irq/irq-affinity.html)
- [/proc/interrupts — man proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html)
- vault: `diagnostics/softirq-ksoftirqd-high.md`, `diagnostics/packet-loss-nic-errors.md`, `diagnostics/high-load-low-cpu.md`
