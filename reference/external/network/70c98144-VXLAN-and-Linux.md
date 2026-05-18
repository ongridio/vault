---
title: VXLAN and Linux
source: https://vincent.bernat.ch/en/blog/2017-vxlan-linux
kind: external
domain: network
author: Vincent Bernat
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, network]
---

> [!info] External article · imported reference
> Source: [vincent.bernat.ch](https://vincent.bernat.ch/en/blog/2017-vxlan-linux)
> Author: Vincent Bernat
> License: cc-by-sa
> Fetched: 2026-05-18

# VXLAN and Linux

**VXLAN** is an **overlay network** to carry Ethernet traffic over an
existing (highly available and scalable) IP network while accommodating
a very large number of tenants. It is defined in [RFC 7348](https://www.rfc-editor.org/rfc/rfc7348 "RFC 7348: Virtual eXtensible Local Area Network (VXLAN)").

Starting from Linux 3.12, the VXLAN implementation is quite complete
as both **multicast** and **unicast** are supported as well as IPv6
and IPv4. Let’s explore the various methods to configure it.

VXLAN deployment example

To illustrate our examples, we use the following setup:

- an **underlay IP network** (highly available and scalable, possibly the Internet);
- three Linux bridges acting as **VXLAN tunnel endpoints** (VTEP); and
- four **servers** believing they share a common Ethernet segment.

A VXLAN tunnel extends the individual Ethernet segments across the
three bridges, providing a unique (virtual) Ethernet segment. From one
host (e.g. `H1`), we can reach directly all the other hosts in the
virtual segment:

```
$ ping -c10 -w1 -t1 ff02::1%eth0
PING ff02::1%eth0(ff02::1%eth0) 56 data bytes
64 bytes from fe80::5254:33ff:fe00:8%eth0: icmp_seq=1 ttl=64 time=0.016 ms
64 bytes from fe80::5254:33ff:fe00:b%eth0: icmp_seq=1 ttl=64 time=4.98 ms (DUP!)
64 bytes from fe80::5254:33ff:fe00:9%eth0: icmp_seq=1 ttl=64 time=4.99 ms (DUP!)
64 bytes from fe80::5254:33ff:fe00:a%eth0: icmp_seq=1 ttl=64 time=4.99 ms (DUP!)

--- ff02::1%eth0 ping statistics ---
1 packets transmitted, 1 received, +3 duplicates, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.016/3.745/4.991/2.152 ms
```

# Basic usage

The reference deployment for VXLAN is to use an IP multicast group to
join the other VTEPs:

```
# ip -6 link add vxlan100 type vxlan \
>   id 100 \
>   dstport 4789 \
>   local 2001:db8:1::1 \
>   group ff05::100 \
>   dev eth0 \
>   ttl 5
# brctl addbr br100
# brctl addif br100 vxlan100
# brctl addif br100 vnet22
# brctl addif br100 vnet25
# brctl stp br100 off
# ip link set up dev br100
# ip link set up dev vxlan100
```

The above commands create a new interface acting as a VXLAN tunnel
endpoint, named `vxlan100`, and put it in a bridge with some regular
interfaces.[1](#sidenote-bridges) Each VXLAN segment is associated with a 24-bit
segment ID, the *VXLAN Network Identifier* (VNI). In our example, the
default VNI is specified with `id 100`.

1

This is one possible implementation. The bridge is only
needed if you require some form of source-address learning for local
interfaces. Another strategy is to use [MACVLAN](https://archive.today/2017/https://hicu.be/bridge-vs-macvlan "Hi Cube: Bridge vs Macvlan") interfaces. ❦

When VXLAN was first implemented in Linux 3.7, the UDP port to use was
not defined. Several vendors were using 8472 and Linux took the same
value. To avoid breaking existing deployments, this is still the
default value. Therefore, if you want to use the IANA-assigned port,
you need to explicitly set it with `dstport 4789`.

As we want to use multicast, we have to specify a multicast group to
join (`group ff05::100`), as well as a physical device (`dev
eth0`). With multicast, the default TTL is 1. If your multicast
network leverages some routing, you’ll have to increase the value a
bit, like here with `ttl 5`.

The `vxlan100` device acts as a bridge device with remote VTEPs as
virtual ports:

- it sends **broadcast, unknown unicast, and multicast** (BUM) frames
  to all VTEPs using the multicast group; and
- it discovers the association from Ethernet MAC addresses to VTEP IP
  addresses using **source-address learning**.

The following figure summarizes the configuration, with the FDB of the
Linux bridge (learning local MAC addresses) and the FDB of the VXLAN
device (learning distant MAC addresses):

VXLAN device attached to a Linux bridge and communicating with two remote VTEPs

The FDB of the VXLAN device can be observed with the `bridge`
command. If the destination MAC is present, the frame is sent to the
associated VTEP (unicast). The all-zero address is only used when a
lookup for the destination MAC fails.

```
# bridge fdb show dev vxlan100 | grep dst
00:00:00:00:00:00 dst ff05::100 via eth0 self permanent
50:54:33:00:00:0b dst 2001:db8:3::1 self
50:54:33:00:00:08 dst 2001:db8:1::1 self
```

If you are interested to get more details on how to set up a multicast
network and build VXLAN segments on top of it, see my
“[Network virtualization with VXLAN](/en/blog/2012-multicast-vxlan "Network virtualization with VXLAN")” article.

# Without multicast

Using VXLAN over a multicast IP network has several benefits:

- automatic discovery of other VTEPs sharing the same multicast
  group
- good bandwidth usage (packets are replicated as late as
  possible)
- decentralized and controller-less design[2](#sidenote-decentralized)

2

The underlay multicast network may still need some
central components, like rendez-vous points for PIM-SM
protocol. Fortunately, it’s possible to make them highly available
and scalable (e.g. with Anycast-RP, [RFC 4610](https://www.rfc-editor.org/rfc/rfc4610 "RFC 4610: Anycast-RP Using Protocol Independent Multicast (PIM)")). ❦

However, multicast is not available everywhere and managing it at
scale can be difficult. In Linux 3.8, the DOVE extensions have been
added to the VXLAN implementation, removing the dependency on
multicast.

## Unicast with static flooding

We can replace multicast by head-end replication of BUM frames to a
statically configured list of remote VTEPs:[3](#sidenote-patch1)

3

For this example and the following ones,
a [patch is needed](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/commit/?id=97d564b90ccb1e4a3c756d9caae161f55b2b63a2 "vxlan: use preferred address family when neither group or remote is specified") for the `ip` command (included
in 4.11) to use IPv6 for transport. In the meantime, here is a
quick workaround:

```
# ip -6 link add vxlan100 type vxlan \
>   id 100 \
>   dstport 4789 \
>   local 2001:db8:1::1 \
>   remote 2001:db8:2::1
# bridge fdb append 00:00:00:00:00:00 \
>   dev vxlan100 dst 2001:db8:3::1
```

❦

```
# ip -6 link add vxlan100 type vxlan \
>   id 100 \
>   dstport 4789 \
>   local 2001:db8:1::1
# bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 2001:db8:2::1
# bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 2001:db8:3::1
```

The VXLAN is defined without a remote multicast group. Instead, all
the remote VTEPs are associated with the all-zero address: a BUM frame
will be duplicated to all these destinations. The VXLAN device will
still learn remote addresses automatically using source-address
learning.

It is a very simple solution. With a bit of automation, you can keep
the default FDB entries up-to-date easily. However, the host will have
to duplicate each BUM frame (head-end replication) as many times as
there are remote VTEPs. This is quite reasonable if you have a dozen
of them. This may become out-of-hand if you have thousands of them.

[Cumulus vxfld daemon](https://github.com/vincentbernat/vxfld "VXFLD: VXLAN BUM Flooding Suite") is an example of this strategy (in the
head-end replication mode). “[Keepalived and unicast over multiple
interfaces](/en/blog/2020-keepalived-unicast-vxlan)” shows another usage.

## Unicast with static L2 entries

When the associations of MAC addresses and VTEPs are known, it is
possible to pre-populate the FDB and disable learning:

```
# ip -6 link add vxlan100 type vxlan \
>   id 100 \
>   dstport 4789 \
>   local 2001:db8:1::1 \
>   nolearning
# bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 2001:db8:2::1
# bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 2001:db8:3::1
# bridge fdb append 50:54:33:00:00:09 dev vxlan100 dst 2001:db8:2::1
# bridge fdb append 50:54:33:00:00:0a dev vxlan100 dst 2001:db8:2::1
# bridge fdb append 50:54:33:00:00:0b dev vxlan100 dst 2001:db8:3::1
```

Thanks to the `nolearning` flag, source-address learning is
disabled. Therefore, if a MAC is missing, the frame will always be
sent using the all-zero entries.

The all-zero entries are still needed for broadcast and multicast
traffic (e.g. ARP and IPv6 neighbor discovery). This kind of setup
works well to provide virtual L2 networks to virtual machines (no L3
information available). You need some glue to update the FDB entries.

[BGP EVPN](https://www.rfc-editor.org/rfc/rfc7432 "RFC 7432: BGP MPLS-Based Ethernet VPN") with [FRR](https://github.com/FRRouting/frr "The FRR Protocol Suite, forked from Quagga") is an example of this strategy (see
“[VXLAN: BGP EVPN with FRR](/en/blog/2017-vxlan-bgp-evpn "VXLAN: BGP EVPN with FRR")” for additional information).

## Unicast with static L3 entries

In the previous example, we had to keep the all-zero entries for ARP
and IPv6 neighbor discovery to work correctly. However, Linux can
answer to neighbor requests on behalf of the remote
nodes.[4](#sidenote-patch2) When this feature is enabled, the default entries are
not needed anymore (but you could keep them):

```
# ip -6 link add vxlan100 type vxlan \
>   id 100 \
>   dstport 4789 \
>   local 2001:db8:1::1 \
>   nolearning \
>   proxy
# ip -6 neigh add 2001:db8:ff::11 lladdr 50:54:33:00:00:09 dev vxlan100
# ip -6 neigh add 2001:db8:ff::12 lladdr 50:54:33:00:00:0a dev vxlan100
# ip -6 neigh add 2001:db8:ff::13 lladdr 50:54:33:00:00:0b dev vxlan100
# bridge fdb append 50:54:33:00:00:09 dev vxlan100 dst 2001:db8:2::1
# bridge fdb append 50:54:33:00:00:0a dev vxlan100 dst 2001:db8:2::1
# bridge fdb append 50:54:33:00:00:0b dev vxlan100 dst 2001:db8:3::1
```

This setup eliminates head-end replication. However, protocols relying
on multicast won’t work either. With some automation, this is a setup
that should work well with containers: if there is a registry keeping
a list of all IP and MAC addresses in use, a program could listen to
it and adjust the FDB and the neighbor tables.

The VXLAN backend of [Docker’s libnetwork](https://github.com/moby/libnetwork "libnetwork - networking for containers") is an example of
this strategy (but it also uses the next method).

## Unicast with dynamic L3 entries

Linux can also notify a program that an (L2 or L3) entry is missing. The
program queries some central registry and dynamically adds the
requested entry. However, for L2 entries, notifications are issued
only if:

- the destination MAC address is not known;
- there is no all-zero entry in the FDB; and
- the destination MAC address is not a multicast or broadcast one.

These limitations prevent us from doing a “unicast with dynamic L2 entries”
scenario.

First, let’s create the VXLAN device with the `l2miss` and `l3miss`
options:[5](#sidenote-patch3)

5

You have to [apply an IPv6-related patch](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=8f48ba71ede12231d5d63fdd34bd8ce7908a3377 "vxlan: use appropriate family on L3 miss") to the
kernel (included in 4.12) to get appropriate notifications for
missing IPv6 addresses. ❦

```
ip -6 link add vxlan100 type vxlan \
   id 100 \
   dstport 4789 \
   local 2001:db8:1::1 \
   nolearning \
   l2miss \
   l3miss \
   proxy
```

Notifications are sent to programs listening to an `AF_NETLINK` socket
using the `NETLINK_ROUTE` protocol. This socket needs to be bound to
the `RTNLGRP_NEIGH` group. The following is doing exactly that and
decodes the received notifications:

```
# ip monitor neigh dev vxlan100
miss 2001:db8:ff::12 STALE
miss lladdr 50:54:33:00:00:0a STALE
```

The first notification is about a missing neighbor entry for the
requested IP address. We can add it with the following command:

```
ip -6 neigh replace 2001:db8:ff::12 \
    lladdr 50:54:33:00:00:0a \
    dev vxlan100 \
    nud reachable
```

The entry is not permanent so that we don’t need to delete it when it
expires. If the address becomes stale, we will get another
notification to refresh it.

Once the host receives our proxy answer for the neighbor discovery
request, it can send a frame with the MAC we gave as a destination.
The second notification is about the missing FDB entry for this MAC
address. We add the appropriate entry with the following
command:[6](#sidenote-sooner)

6

Directly adding the entry after the first notification
would have been smarter to avoid unnecessary retransmissions. ❦

```
bridge fdb replace 50:54:33:00:00:0a \
    dst 2001:db8:2::1 \
    dev vxlan100 dynamic
```

The entry is not permanent either as it would prevent the MAC to
migrate to the local VTEP (a dynamic entry cannot override a permanent
entry).

This setup works well with containers and a global registry. However,
there is a small latency penalty for the first connections. Moreover,
multicast and broadcast won’t be available in the underlay
network. The VXLAN backend for [flannel](https://github.com/flannel-io/flannel "flannel: a network fabric for containers, designed for Kubernetes"), a network fabric for
*Kubernetes*, is an example of this strategy.

# Decision

There is no one-size-fits-all solution.

You should consider the **multicast** solution if:

- you are in an environment where multicast is available;
- you are ready to operate (and scale) a multicast network;
- you need multicast and broadcast inside the virtual segments; and
- you don’t have L2/L3 addresses available beforehand.

The scalability of such a solution is pretty good if you take care of
not putting all VXLAN interfaces into the same multicast group
(e.g. use the last byte of the VNI as the last byte of the multicast
group).

When multicast is not available, another generic solution is **BGP
EVPN**: BGP is used as a controller to ensure the distribution of the
list of VTEPs and their respective FDBs. As mentioned earlier, an
implementation of this solution is [FRR](https://github.com/FRRouting/frr "The FRR Protocol Suite, forked from Quagga"). I explore this option in a
separate post: [VXLAN: BGP EVPN with FRR](/en/blog/2017-vxlan-bgp-evpn "VXLAN: BGP EVPN with FRR").

If you operate in a container-like environment where L2/L3 addresses
are known beforehand, a solution using **static and/or dynamic L2 and
L3 entries** based on a central registry and no source-address
learning would also fit the bill. This provides a more security-tight
solution (bound resources, MiTM attacks dampened down, inability to
amplify bandwidth usage through excessive broadcast). Various
environment-specific solutions are available[7](#sidenote-examples) or you can
build your own.

# Other considerations

Independently of the chosen strategy, here are a few important points
to keep in mind when implementing a VXLAN overlay.

## Isolation

While you may expect VXLAN interfaces to only carry L2 traffic, Linux
doesn’t disable IP processing. If the destination MAC is a local one,
Linux will route or deliver the encapsulated IP packet. Check my post
about the [proper isolation of a Linux bridge](/en/blog/2017-linux-bridge-isolation "Proper isolation of a Linux bridge").

## Encryption

VXLAN enforces isolation between tenants, but the traffic is
unencrypted. The most direct solution to provide encryption is to use
*IPsec*. Some container-based solutions may come with IPsec support
out-of-the-box (notably [Docker’s libnetwork](https://github.com/moby/libnetwork "libnetwork - networking for containers"), but [flannel](https://github.com/flannel-io/flannel "flannel: a network fabric for containers, designed for Kubernetes") has
a plan for it too). This is quite important for deployment over a
public cloud.

## Overhead

The format of a VXLAN-encapsulated frame is the following:

VXLAN encapsulation inside a UDP datagram (additional overhead: 50 bytes)

VXLAN adds a fixed overhead of 50 bytes. If you also use IPsec, the
overhead depends on many factors. In transport mode, with AES and
SHA256, the overhead is 56 bytes. With NAT traversal, this is 64 bytes
(additional UDP header). In tunnel mode, this is 72 bytes.

Some users will expect to be able to use an Ethernet MTU of 1500 for
the overlay network. Therefore, the underlay MTU should be
increased. If it is not possible, ensure the inner MTU (inside the
containers or the virtual machines) is correctly decreased.[8](#sidenote-pmtu)

## IPv6

While all the examples above are using IPv6, the ecosystem is not
quite ready yet. The multicast L2-only strategy works fine with IPv6
but every other scenario currently needs some patches ([1](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/commit/?id=97d564b90ccb1e4a3c756d9caae161f55b2b63a2 "vxlan: use preferred address family when neither group or remote is specified"),
[2](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=f1fb08f6337ca9e3af371a7994b91a5786ba93f9 "vxlan: fix ND proxy when skb doesn't have transport header offset"), [3](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=8f48ba71ede12231d5d63fdd34bd8ce7908a3377 "vxlan: use appropriate family on L3 miss"), [4](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git/commit/?id=8bff3685a4bbf175a96bc6a528f13455d8d38244 "vxlan: fix the issue that neigh proxy blocks all icmpv6 packets")).

Update (2020-11)

From Linux 4.13, all the mentioned issues are
solved. IPv6 is also more well-tested as Cumulus promotes IPv6 as the
transport and signaling protocol for their own solutions.

On top of that, IPv6 may not have been implemented in VXLAN-related
tools:

## Multicast

Linux VXLAN implementation doesn’t support IGMP snooping. Multicast
traffic will be broadcasted to all VTEPs unless multicast MAC
addresses are inserted into the FDB.
