---
title: Packet Loss & NIC-Level Drops/Errors
kind: howto
tags: [network, nic, packet-loss, ethtool, ring-buffer, drops, offload]
applies_to: [edge, manager]
---

# Packet Loss & NIC-Level Drops/Errors

Use when throughput is low / retransmits are high and you've already
ruled out the application — drops are happening at or below the driver.
**Distinguish "the wire is lossy" (counters on errors/CRC) from "the host
is dropping" (ring buffer / qdisc / socket backlog overflow).** The fix
differs: escalate to network vs. tune the host.

| Where counted | Meaning |
|---|---|
| `rx_errors` / `rx_crc_errors` | bad frames on the wire — cabling/SFP/duplex |
| `rx_dropped` / `rx_fifo_errors` | NIC ring full — host not draining fast enough |
| `rx_missed_errors` | NIC FIFO overflow — PCIe/IRQ can't keep up |
| qdisc `dropped` (tc) | egress queue overflow — shaping/backpressure |
| socket `RcvbufErrors` (UDP) | app not reading fast enough |

## Step 1 — Read the NIC counters

```bash
ip -s link show <iface>             # high-level rx/tx errors, dropped
ethtool -S <iface> | grep -Ei 'err|drop|miss|fifo|crc|nodesc|discard'
```

Take a sample, wait 30s, diff. A counter that's large-but-static is
historical; one that's *increasing* is your live problem.

## Step 2 — Link health (the physical-layer drops)

```bash
ethtool <iface> | grep -E 'Speed|Duplex|Link detected'
ethtool <iface> | grep -i 'auto-negotiation'
```

`Half` duplex on a `Full` peer, or a speed mismatch, produces CRC/error
storms. `Link detected: no` flapping = cable/SFP. These are physical —
escalate, don't tune.

## Step 3 — Ring buffer + host-drain (the host-side drops)

```bash
ethtool -g <iface>                  # current vs max RX/TX ring
# If rx_dropped rises and RX ring < max, bump it:
ethtool -G <iface> rx 4096
# Are softirqs keeping up? ksoftirqd hot = NIC out-running the CPU
mpstat -P ALL 1 3 | grep -i soft
```

`rx_dropped` rising with a small ring buffer = classic host can't drain
fast enough → enlarge the ring and/or fix softirq saturation (see
`diagnostics/softirq-ksoftirqd-high.md`).

## Step 4 — Offload + qdisc sanity

```bash
ethtool -k <iface> | grep -E 'gro|gso|tso|rx-checksumming'   # offloads
tc -s qdisc show dev <iface>        # egress drops/overlimits
```

Buggy offload (rare) causes checksum/drop oddities — toggling
`gro`/`tso` off is a quick test. Sustained qdisc drops = egress shaping
or NIC TX saturation (see `diagnostics/bandwidth-saturation.md`).

## Decision tree

| Signal | Action |
|---|---|
| `rx_crc_errors` / `rx_errors` rising | physical — cable/SFP/duplex; escalate to network |
| `rx_dropped`, ring < max | enlarge RX ring (`ethtool -G`) |
| `rx_missed`, ksoftirqd hot | softirq/IRQ saturation — RPS/RSS, IRQ affinity |
| qdisc `dropped` rising | egress saturation/shaping — see bandwidth playbook |
| UDP `RcvbufErrors` | app too slow / `rmem` too small |

## References

- [ethtool(8)](https://man7.org/linux/man-pages/man8/ethtool.8.html)
- [Monitoring and Tuning the Linux Networking Stack — Packagecloud](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
- vault: `diagnostics/softirq-ksoftirqd-high.md`, `diagnostics/bandwidth-saturation.md`, `diagnostics/tcp-retransmit-loss.md`
