---
title: MTU Mismatch / PMTU Black Hole
kind: howto
tags: [network, mtu, pmtud, fragmentation, vpn, overlay, icmp]
applies_to: [edge, manager]
---

# MTU Mismatch / PMTU Black Hole

Use for the maddening "small requests work, large ones hang" pattern: TCP
connects fine (SYN/handshake are tiny), `ping` works, but a big POST,
TLS with a large cert, or a bulk transfer stalls. A path can't carry your
packet size and the ICMP "fragmentation needed" that should fix it is
being dropped — a **PMTU black hole**. Classic on VPN/tunnel/overlay
links where MTU < 1500.

| Symptom | Probable cause |
|---|---|
| Small ok, large hangs | MTU mismatch + PMTUD broken (ICMP filtered) |
| Only over VPN / tunnel / overlay | encapsulation shrinks usable MTU |
| TLS handshake hangs at certificate | the (large) cert packet exceeds path MTU |
| Works after lowering MSS | confirms MTU/MSS is the issue |

## Step 1 — Find the real path MTU

```bash
# Increase payload until it fails (DF set, no fragmentation allowed):
ping -M do -s 1472 <dest>      # 1472 + 28 hdr = 1500; lower until it succeeds
ping -M do -s 1400 <dest>
# tracepath discovers PMTU per hop
tracepath <dest>
```

The largest `-s` that succeeds + 28 = your path MTU. If 1472 fails but
1400 works, the path MTU is ~1428 — something in the middle is smaller
than 1500.

## Step 2 — Compare interface MTUs along your side

```bash
ip link show | grep -E 'mtu'           # local ifaces + tunnels
# Tunnels/overlays subtract overhead: WireGuard ~1420, VXLAN ~1450, IPsec varies
```

A tunnel/overlay interface at 1500 (not reduced for its own header) sends
packets that can't traverse the encapsulated path → black hole.

## Step 3 — Confirm PMTUD is being eaten

PMTUD relies on ICMP type 3 code 4 ("fragmentation needed"). Overzealous
firewalls drop all ICMP, so the sender never learns to shrink → packets
silently vanish. Watch for the absence:

```bash
sudo tcpdump -nn -i any 'icmp and icmp[icmptype]==3'   # should see frag-needed on big sends
```

No frag-needed arriving while big packets disappear = ICMP is filtered
somewhere → black hole.

## Step 4 — Fixes

- **Right-size the interface MTU** on tunnels/overlays to account for
  encapsulation overhead (set it below the path MTU).
- **MSS clamping** on the gateway forces TCP to advertise a smaller MSS,
  sidestepping PMTUD entirely:
  ```bash
  iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu
  ```
- **Allow ICMP frag-needed** through firewalls so PMTUD works.

## Decision tree

| Signal | Action |
|---|---|
| big fails / small ok over tunnel | lower the tunnel/overlay interface MTU |
| works after MSS clamp | clamp MSS to PMTU on the gateway |
| no ICMP frag-needed seen | unblock ICMP type 3 code 4 on firewalls |
| k8s pod-to-pod large fails | CNI overlay MTU misconfigured — match underlay − overhead |
| only off-site / VPN | provider path MTU < 1500 — clamp + set tunnel MTU |

## References

- [Path MTU Discovery — RFC 1191 (concept)](https://www.rfc-editor.org/rfc/rfc1191)
- [PMTUD black holes — Cloudflare blog](https://blog.cloudflare.com/path-mtu-discovery-in-practice/)
- vault: `diagnostics/network-slow-cn.md`, `diagnostics/tcp-retransmit-loss.md`, `diagnostics/tls-handshake-failure.md`, `systems/network/tcp-stack.md`
