---
title: Open vSwitch — Why Isn't a Flow Matching
kind: howto
tags: [ovs, openvswitch, flow-table, vxlan, vlan, mac-learning]
applies_to: [edge]
---

# Open vSwitch — Why Isn't a Flow Matching

Use when packets enter an OVS bridge and don't come out the expected
port, or hit the controller / "normal" fallback instead of a specific
flow. OVS has **three** tables in front of you (flow / fdb / userspace
controller), all silent on miss — debugging means asking each one in
turn.

## Step 1 — Quick topology recall

```bash
ovs-vsctl show                          # bridges, ports, controllers
ovs-ofctl show br0                      # OF ports + their numbers
ovs-vsctl list bridge br0 | grep -E 'flow_tables|controller|fail_mode'
```

Note three things before going deeper:
1. **fail_mode**: `secure` (no controller = no traffic) vs `standalone` (no controller = act as L2 switch). A controller dying turns `secure` bridges into black holes.
2. **OF port numbers** — flows reference these, not Linux ifindex.
3. **Controller present?** A flow `actions=CONTROLLER:65535` means "I don't know, ask SDN controller". Spike of those = controller dead or rule missing.

## Step 2 — Watch packets arriving on the bridge

```bash
# Show OF stats per port (rx_packets, tx_packets, drops)
ovs-ofctl dump-ports br0

# If rx is increasing on the ingress port but tx isn't increasing on
# the expected egress port, the flow isn't producing the action you
# expect. Dump the flow table:
ovs-ofctl dump-flows br0 --rsort=priority
```

For VLAN-tagged inputs add `--names` so symbolic port names print:

```bash
ovs-ofctl --names dump-flows br0
```

## Step 3 — Trace what would happen for a specific packet

```bash
# Build a synthetic packet matching what you expect to see, then ask
# OVS to trace it through the flow tables. NO real packet is sent.
ovs-appctl ofproto/trace br0 \
  in_port=eth0,dl_src=aa:bb:cc:dd:ee:ff,dl_dst=11:22:33:44:55:66,dl_vlan=10
```

Output shows: matched rule(s), table jumps, final actions. If trace
gives "no match" but you have a low-priority `actions=NORMAL` flow,
NORMAL mode then does its own MAC learning — go to Step 4.

## Step 4 — Check the MAC learning (FDB) table

```bash
ovs-appctl fdb/show br0
# columns: port VLAN MAC age
```

NORMAL mode and many "L2 switch" flows depend on the FDB knowing which
port a MAC sits behind. Empty / missing entry → packet floods (which
might look like "isn't reaching the right port"). Common reasons:

| FDB symptom | Cause |
|---|---|
| Empty FDB | Recent restart / `ovs-appctl fdb/flush` was called |
| MAC on wrong port | Loop / asymmetric path / migration without GARP |
| Entry flapping rapidly | L2 loop — check STP, run `ovs-appctl stp/show` if enabled |
| MAC missing on tunnel port | Tunnel down — see Step 5 |

Manually inject a learning hint:

```bash
ovs-appctl fdb/add br0 aa:bb:cc:dd:ee:ff 10 vxlan0
```

## Step 5 — Tunnel ports (VXLAN/GRE/Geneve) silently dropping

```bash
# Tunnel interface status
ovs-vsctl list interface vxlan0 | grep -E 'admin_state|link_state|options'
ovs-vsctl get Interface vxlan0 status:tunnel_egress_iface
ovs-vsctl get Interface vxlan0 statistics
```

VXLAN-specific gotchas:
- Underlying interface MTU too small. VXLAN adds 50 bytes — set bridge
  MTU 1450 if underlay is 1500, otherwise large packets silently drop.
- Underlay route missing or asymmetric. Check `ip route get <tunnel
  remote_ip>` on both ends.
- Outer ToS / DSCP rewrites by intermediate hops can break ECN-marked
  inner flows.

## Step 6 — Datapath kernel cache

OVS keeps a per-flow datapath microflow cache (visible via `ovs-dpctl
dump-flows`). If a flow shows up in `ofctl dump-flows` but `dpctl`
shows zero matches, the kernel module isn't seeing the packets at all
— check the netfilter chain doesn't drop pre-OVS:

```bash
ovs-dpctl dump-flows | head -20         # what kernel actually saw
iptables -L FORWARD -n -v               # drops before OVS?
nft list ruleset | grep -E 'drop|reject' | head
```

## Decision tree

| Observed | Likely root cause |
|---|---|
| `dump-ports rx > 0` but `tx_<egress> == 0`, flows match in `ofproto/trace` | Flow action is wrong (output to wrong port?) |
| `ofproto/trace` returns "no match", `actions=NORMAL` exists | FDB miss → flood — check fdb/show |
| Trace OK, dpctl shows nothing | Pre-OVS netfilter drop |
| FDB empty / flapping | Migration / loop / STP issue |
| Tunnel port `link_state=down` | Underlay route or firewall |
| Spike of `actions=CONTROLLER` | SDN controller missing rule for this flow class |

## References

- [Open vSwitch FAQ — official](https://docs.openvswitch.org/en/latest/faq/)
- [`ovs-appctl ofproto/trace` man page](https://man7.org/linux/man-pages/man8/ovs-appctl.8.html)
- vault: `systems/network/tcp-stack.md` (for L4 issues post-OVS)
