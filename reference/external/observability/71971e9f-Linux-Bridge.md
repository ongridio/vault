---
title: Linux Bridge
source: https://goyalankit.com/blog/linux-bridge
kind: external
domain: observability
original_date: 2017-05-07
fetched_at: 2026-05-16
bookmark_title: Linux Bridge - how it works
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[goyalankit.com](https://goyalankit.com/blog/linux-bridge)
> 原始日期：2017-05-07
> 抓取日期：2026-05-16

# Linux Bridge

# Linux Bridge - how it works

May 7, 2017

Linux bridge is a layer 2 virtual device that on its own cannot receive or transmit anything unless you bind one or more real devices to it. source.

As Anatomy of a Linux bridge puts it, bridge mainly consists of four major components:

**Set of network ports (or interfaces)**: used to forward traffic between end switches to other hosts in the network.**A control plane**: used to run Spanning Tree Protocol (STP) that calculates minimum spanning tree, preventing loops from crashing the network.**A forwarding plane**: used to process incoming input frames from the ports, forward them to the network port by making a forwarding decision based on the MAC learning database.**MAC learning database**: used to keep track of the host locations in the LAN.

For each unicast mac address, bridge maintains a mac learning database to decide which ports to forward based on MAC addresses, and if it can’t find an entry for a given mac address, it will broadcast the frame to all ports except the one where it received the frame from.

There are three main configuration subsystems to do bridges:

**ioctl**: This interface is used to create/destroy bridges and add/remove interfaces to/from a bridge.**sysfs**: Management of bridge and port specific parameters.**netlink**: Asynchronous queue based communication that uses**AF_NETLINK**address family, can also be used to interact with bridge.

In this article, we only talk about **ioctl**.

## Creating a bridge

Bridge can be created using `ioctl`

command `SIOCBRADDBR`

; as can be seen by `brctl`

utility provided by bridge-utils.

Note that there is no device at this point to handle the ioctl command, so the ioctl command is handled by a stub method: `br_ioctl_deviceless_stub`

, which in turn calls `br_add_bridge`

. This method calls `alloc_netdev`

, which is a macro that eventually calls `alloc_netdev_mqs`

.

`alloc_netdev`

also initializes the new netdevice using the `br_dev_setup`

. This also includes setting up the bridge specific `ioctl handler`

. If you look at the handler code, it handles ioctl command to add/delete interfaces.

## Adding an interface

As it can be seen in `br_dev_ioctl`

, bridge can be created using `ioctl`

command `SIOCBRADDIF`

. To confirm:

`br_add_if`


The `br_add_if`

method creates and sets up the new interface/port for the bridge by allocating a new `net_bridge_port`

object. The object initialization is particularly interesting, as it sets the interface to receive all traffic, adds the network interface address for the new interface to the forwarding database as the local entry and attaches the interface as the slave to the bridge device.

Some things worth noting in `br_add_if`

:

- Only ethernet like devices can be added to bridge, as bridge is a layer 2 device.
- Bridges cannot be added to a bridge.
- New interface is set to promiscuous mode:
`dev_set_promiscuity(dev, 1)`


The promiscuous mode can be confirmed from kernel logs.

Finally, `br_add_if`

method calls `netdev_rx_handler_register`

, that sets the `rx_handler`

of the interface to `br_handle_frame`


After this method finishes, you have an interface (or port) in bridge.

## Frame Processing

Frame processing starts with device-independent network code, in `__netif_receive_skb`

which calls the `rx_handler`

of the interface, that was set to `br_handle_frame`

at the time of adding the interface to bridge.

The `br_handle_frame`

does the initial processing and any address with prefix `01-80-C2-00-00`

is a control plane address, that may need special processing. From the comments in `br_handle_frame`

:

In the method, note that stp messages are either passed to upper layers or forwarded if STP is enabled on the bridge or disabled respectively. Finally if a forwarding decision is made, the packet is passed to `br_handle_frame_finish`

, where the actual forwarding happens.

Here’s the highly truncated version of `br_handle_frame_finish`

:

As you can see in above snippet of `br_handle_frame_finish`

,

- An entry in forwarding database is updated for the source of the frame.
- (not in the above snippet) If the destination address is a multicast address, and if the multicast is disabled, the packet is dropped. Or else message is received using
`br_multicast_rcv`

- Now if the promiscuous mode is on, packet will be delivered locally, irrespective of the destination.
- For a unicast address, we try to determine the port using the forwarding database (
`__br_fdb_get`

). - If the destination is local, then
`skb`

is set to null i.e., packet will not be forwarded. - If the destination is not local, then based on if we found an entry in forwarding database, either the frame is forwarded (
`br_forward`

) or flooded to all ports (`br_flood_forward`

). - Later, packet is delivered locally (
`br_pass_frame_up`

) if needed (based on either the current host being the destination or the net device being in promiscuous mode).

`br_forward`

method either clones and then deliver (if it is also to be delivered locally, by calling `deliver_clone`

), or directly forwards the message to the intended destination interface by calling `__br_forward`

.

`bt_flood_forward`

forwards the frame on each interface by iterating through the list in `br_flood`

method.

# Conclusion

Bridges can be used to create various different network topologies and it’s important to understand how they work. I have seen bridges being used with containers where they are used to provide networking in network namespaces along with `veth`

devices. In fact the default networking in docker is provided using bridge.

This is all for now, hopefully this was useful. This was mainly based on the excellent paper Anatomy of a Linux bridge and my own reading of linux kernel code. I’d appreciate any feedback, or comments.

Ah, the wonderful world of bridges.

**Update July 12, 2017**: Added third way to communicate with bridge. *Thanks to @vbernat* from comments.

## References:

[1] Anatomy of a Linux bridge

[2] Understanding Linux Networking Internals - Christian Benvenuti

[3] Linux kernel v3.10.105 source code