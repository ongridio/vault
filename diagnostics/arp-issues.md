---
title: ARP Problems — Duplicate IP, Stale Entries, ARP Storms
kind: howto
tags: [network, arp, l2, duplicate-ip, neighbor, gratuitous-arp]
applies_to: [edge, manager]
---

# ARP Problems — Duplicate IP, Stale Entries, ARP Storms

Use for L2 weirdness: intermittent connectivity to a host on the same
subnet, "works for a while then stops", connections landing on the *wrong*
machine, or a flood of ARP traffic. ARP maps IP→MAC on the local segment;
when two hosts claim one IP, or the neighbor cache goes stale/full, packets
go to the wrong (or no) MAC.

| Symptom | Probable cause |
|---|---|
| Intermittent, flaps between working/not | duplicate IP — two MACs answering for one IP |
| Reaches wrong host sometimes | same — switch learns whichever MAC last replied |
| Stops after idle, resumes on activity | stale ARP entry / neighbor GC |
| `neighbour table overflow` in dmesg | ARP cache full (many neighbors / scan) |

## Step 1 — Inspect the neighbor table

```bash
ip neigh show                        # IP → MAC, state (REACHABLE/STALE/FAILED)
ip neigh show <ip>                   # the specific peer
# FAILED / INCOMPLETE = no ARP reply (host down / L2 broken / filtered)
```

`FAILED`/`INCOMPLETE` for a host that should be up = it's not answering
ARP (down, wrong VLAN, or L2 isolation). Frequent `STALE`→re-resolve is
normal; constant churn is not.

## Step 2 — Hunt a duplicate IP

```bash
# Does more than one MAC claim the IP?
arping -D -I <iface> -c 3 <ip>       # duplicate address detection
sudo tcpdump -nn -i <iface> arp and host <ip>   # watch who replies
```

Two different MACs replying for one IP = duplicate IP (a misconfigured
host, a forgotten static, a DHCP/static collision, or a failover VIP
gone wrong). This is the top cause of "flaps between working and not".

## Step 3 — Stale entries / GC tuning

```bash
# Neighbor GC thresholds — small tables overflow on busy/large segments
sysctl net.ipv4.neigh.default.gc_thresh1 gc_thresh2 gc_thresh3
dmesg -T | grep -i 'neighbour table overflow'
```

`neighbour table overflow` (common on nodes with many pods/peers, e.g.
large k8s nodes) means the ARP cache is too small — raise `gc_thresh*`.

## Step 4 — ARP storms / gratuitous ARP

A broadcast storm of ARP (often a loop, or a failover flapping and
sending gratuitous ARPs repeatedly) saturates the segment. Watch ARP
volume with tcpdump; a host re-announcing constantly (flapping VIP, split
HA) is the source — stabilize the failover.

## Decision tree

| Signal | Action |
|---|---|
| two MACs answer one IP | duplicate IP — find + fix the colliding host/VIP |
| `FAILED`/`INCOMPLETE` for a live host | L2/VLAN/isolation — check switchport, VLAN |
| `neighbour table overflow` | raise `gc_thresh1/2/3` (big segments / many pods) |
| ARP flood | find the flapping/looping source; stabilize HA failover |
| stale-then-works | usually benign GC; only act if it disrupts traffic |

## References

- [ip-neighbour(8) — man7](https://man7.org/linux/man-pages/man8/ip-neighbour.8.html)
- [arping(8) — duplicate address detection](https://man7.org/linux/man-pages/man8/arping.8.html)
- vault: `diagnostics/network-connectivity.md`, `diagnostics/conntrack-table-full.md`, `systems/network/tcp-stack.md`
