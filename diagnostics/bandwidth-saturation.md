---
title: Network Bandwidth Saturation & Egress Throttling
kind: howto
tags: [network, bandwidth, throughput, qdisc, tc, saturation, egress]
applies_to: [edge, manager]
---

# Network Bandwidth Saturation & Egress Throttling

Use when transfers are slow and you suspect the link, not loss/latency:
throughput plateaus, qdisc drops climb, or a cloud instance hits its NIC
bandwidth cap. **Separate "the link is genuinely full" from "a shaper/cap
is throttling us below the link" from "one flow is hogging it".**

| Symptom | Probable cause |
|---|---|
| Throughput pinned at a round number | provider/instance bandwidth cap |
| qdisc `dropped`/`overlimits` rising | egress queue overflow / shaping |
| One flow consumes all bandwidth | no fairness/QoS; a bulk transfer starves others |
| Saturated only one direction | asymmetric link / upload cap |

## Step 1 — Measure actual throughput + who's using it

```bash
# Live per-interface throughput
sar -n DEV 1 5                       # rxkB/s txkB/s per iface
# Per-process / per-connection bandwidth
iftop -i <iface> 2>/dev/null         # top talkers
nethogs <iface> 2>/dev/null          # bandwidth per process
```

Compare the measured rate to the link's rated speed (`ethtool <iface> |
grep Speed`). Pinned well below the NIC speed = a cap/shaper, not the
physical link.

## Step 2 — qdisc drops + shaping

```bash
tc -s qdisc show dev <iface>         # dropped, overlimits, requeues
tc -s class show dev <iface>         # if HTB/TBF shaping is configured
```

Rising `dropped`/`overlimits` = the egress queue can't keep up or a
configured shaper is actively throttling. If you didn't configure tc, a
provider or k8s bandwidth annotation might have.

## Step 3 — Cloud / instance caps

Cloud NICs have bandwidth allowances tied to instance size (and sometimes
burst credits, like CPU). Sustained transfers hit the baseline cap and
throttle. Check the provider's network-throughput metric for the
instance; if you're at the documented cap, the fix is a bigger instance,
not tuning.

## Step 4 — Fairness & shaping fixes

- **Fair queueing**: `fq_codel` (often default) prevents one bulk flow
  from starving interactive traffic; verify it's the qdisc.
- **Rate-limit the hog**: shape the bulk transfer (tc HTB/TBF, or
  app-level) so it can't consume the whole pipe.
- **Scale the link**: bigger instance / more NICs / bonding if the
  workload legitimately needs more than the link provides.

## Decision tree

| Signal | Action |
|---|---|
| pinned below NIC speed | provider/instance cap — resize, or accept the cap |
| qdisc dropped/overlimits | egress saturated — shape the hog, add bandwidth |
| one flow hogs link | enable fq_codel; rate-limit the bulk transfer |
| NIC `tx` errors too | physical/driver — see `diagnostics/packet-loss-nic-errors.md` |
| burst-then-throttle (cloud) | network credits depleted — bigger instance class |

## References

- [tc(8) & qdisc — man7](https://man7.org/linux/man-pages/man8/tc.8.html)
- [Bufferbloat & fq_codel — bufferbloat.net](https://www.bufferbloat.net/projects/codel/wiki/)
- vault: `diagnostics/packet-loss-nic-errors.md`, `diagnostics/network-slow-cn.md`, `diagnostics/cpu-steal-noisy-neighbor.md`
