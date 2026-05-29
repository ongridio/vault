---
title: High softirq / ksoftirqd CPU (Network Stack Saturation)
kind: howto
tags: [softirq, ksoftirqd, network, rps, rss, irq, cpu]
applies_to: [edge, manager]
---

# High softirq / ksoftirqd CPU (Network Stack Saturation)

Use when `top` shows `ksoftirqd/N` burning a whole core, or `%soft` is
high in `mpstat`, often with one CPU pinned while others idle. Softirqs
process the bottom half of interrupts — mostly NET_RX. A single core
saturated on softirq throttles all packet processing and shows up as
NIC `rx_dropped`/`rx_missed` and latency. **First confirm it's network
softirq and whether it's stuck on one CPU.**

| Symptom | Probable cause |
|---|---|
| One CPU 100% `%soft`, others idle | all NIC IRQs landing on one core (no RSS/RPS) |
| `ksoftirqd` hot under packet flood | RX rate exceeds single-core capacity |
| `%soft` high + NIC `rx_dropped` rising | softirq can't drain the ring fast enough |
| Spikes correlate with traffic bursts | normal-ish; needs spreading, not a bug |

## Step 1 — Confirm it's softirq, and which kind

```bash
mpstat -P ALL 1 3                    # per-CPU %soft — is it one core or all?
cat /proc/softirqs                   # which softirq: NET_RX / NET_TX / TIMER ...
#   sample twice, diff — the column growing fastest is the culprit
```

A lopsided NET_RX column on one CPU = interrupts not spread. TIMER-heavy
is a different story (timer storm, not network).

## Step 2 — Are IRQs pinned to one CPU?

```bash
cat /proc/interrupts | grep -iE "<iface>|eth|ens|enp"   # which CPUs get NIC IRQs
# Is irqbalance running?
systemctl is-active irqbalance
```

All NIC IRQs on CPU0 = the classic single-core bottleneck on multi-queue
NICs that aren't spread.

## Step 3 — Spread the load (RSS / RPS / RFS)

```bash
# Multi-queue NIC: how many RX queues / combined channels
ethtool -l <iface>
ethtool -L <iface> combined N        # use more queues (hardware RSS)
# Software RPS (spread RX softirq across CPUs) when HW queues are limited:
echo  > /sys/class/net/<iface>/queues/rx-0/rps_cpus   # bitmask of CPUs
```

Enabling multiple HW queues (RSS) or RPS distributes NET_RX across cores
so no single `ksoftirqd` saturates.

## Step 4 — Reduce the per-packet cost

- Confirm GRO/GSO offloads are on (`ethtool -k`) so the stack handles
  fewer, larger segments.
- A flood of tiny packets (DDoS, chatty UDP, retransmit storm) raises
  softirq regardless of spreading — treat the source (see
  `diagnostics/tcp-retransmit-loss.md`).

## Decision tree

| Signal | Action |
|---|---|
| One CPU `%soft` 100%, NIC single-queue | enable RPS to spread RX across cores |
| Multi-queue NIC, IRQs on one CPU | spread via more channels + irqbalance/affinity |
| `%soft` high across all cores under flood | reduce packet rate at source; offloads on |
| NET_RX + `rx_dropped` rising | enlarge ring + spread softirq (combine fixes) |
| TIMER softirq dominant | not network — hunt high-res timer abusers |

## References

- [Monitoring and Tuning the Linux Networking Stack: Receiving Data — Packagecloud](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
- [Scaling in the Linux Networking Stack (RSS/RPS/RFS) — kernel.org](https://www.kernel.org/doc/html/latest/networking/scaling.html)
- vault: `diagnostics/packet-loss-nic-errors.md`, `diagnostics/high-load-low-cpu.md`, `systems/network/tcp-stack.md`
