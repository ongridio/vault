---
title: QEMU/KVM Bridged Network with TAP interfaces
source: https://blog.elastocloud.org/2015/07/qemukvm-bridged-network-with-tap.html
kind: external
domain: network
author: Ddiss
original_date: 2015-07-12
fetched_at: 2026-05-16
bookmark_title: Elasticity: QEMU/KVM Bridged Network with TAP interfaces
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.elastocloud.org](https://blog.elastocloud.org/2015/07/qemukvm-bridged-network-with-tap.html)
> 作者：Ddiss
> 原始日期：2015-07-12
> 抓取日期：2026-05-16

# QEMU/KVM Bridged Network with TAP interfaces

This post describes how to plumb the Linux VM directly into a hypervisor network, through the use of a bridge.

Start by creating a bridge on the hypervisor system:

> sudo ip link add br0 type bridge

Clear the IP address on the network interface that you'll be bridging (e.g. eth0).

**Note:**This will disable network traffic on eth0!

> sudo ip addr flush dev eth0Add the interface to the bridge:

> sudo ip link set eth0 master br0

Next up, create a TAP interface:

> sudo ip tuntap add dev tap0 mode tap user $(whoami)The

*user*parameter ensures that the current user will be able to connect to the TAP interface.

Add the TAP interface to the bridge:

> sudo ip link set tap0 master br0

Make sure everything is up:

> sudo ip link set dev br0 up > sudo ip link set dev tap0 up

The TAP interface is now ready for use. Assuming that a DHCP server is available on the bridged network, the VM can now obtain an IP address during boot via:

> qemu-kvm -kernel arch/x86/boot/bzImage \ -initrd initramfs \ -device e1000,netdev=network0,mac=52:55:00:d1:55:01 \ -netdev tap,id=network0,ifname=tap0,script=no,downscript=no \ -append "ip=dhcp rd.shell=1 console=ttyS0" -nographic

The MAC address is explicitly specified, so care should be taken to ensure its uniqueness.

The DHCP server response details are printed alongside network interface configuration. E.g.

[ 3.792570] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX [ 3.796085] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready [ 3.812083] Sending DHCP requests ., OK [ 4.824174] IP-Config: Got DHCP answer from 10.155.0.42, my address is 10.155.0.1 [ 4.825119] IP-Config: Complete: [ 4.825476] device=eth0, hwaddr=52:55:00:d1:55:01, ipaddr=10.155.0.1, mask=255.255.0.0, gw=10.155.0.254 [ 4.826546] host=rocksolid-sles, domain=suse.de, nis-domain=suse.de ...

Didn't get an IP address? There are a few things to check:

- Confirm that the kernel is built with boot-time DHCP client (
*CONFIG_IP_PNP_DHCP=y*) and E1000 network driver (*CONFIG_E1000=y*) support. - Check the
*-device*and*-netdev*arguments specify a valid e1000 TAP interface. - Ensure that
*ip=dhcp*is provided as a kernel boot parameter, and that the DHCP server is up and running.

*Update 20161223:*

*Use 'ip' instead of 'brctl' to manipulate the bridge device - thanks Yagamy Light!*

*Update 20171220:*

*Use 'ip tuntap' instead of 'tunctl' to create the TAP interface - thanks Johannes!*

*Update 20200930:*

*For performance reasons, I strongly recommend using virtio network adapters instead of e1000.*