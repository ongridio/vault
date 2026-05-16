---
title: Native Open vSwitch firewall driver¶
source: https://docs.openstack.org/ocata/networking-guide/config-ovsfwdriver.html
kind: external
domain: network
author: Updated
original_date: 2019-08-23
fetched_at: 2026-05-16
bookmark_title: OpenStack Docs: Native Open vSwitch firewall driver
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[docs.openstack.org](https://docs.openstack.org/ocata/networking-guide/config-ovsfwdriver.html)
> 作者：Updated
> 原始日期：2019-08-23
> 抓取日期：2026-05-16

# Native Open vSwitch firewall driver¶

Note

Experimental feature or incomplete documentation.

Historically, Open vSwitch (OVS) could not interact directly with *iptables*
to implement security groups. Thus, the OVS agent and Compute service use
a Linux bridge between each instance (VM) and the OVS integration bridge
`br-int`

to implement security groups. The Linux bridge device contains
the *iptables* rules pertaining to the instance. In general, additional
components between instances and physical network infrastructure cause
scalability and performance problems. To alleviate such problems, the OVS
agent includes an optional firewall driver that natively implements security
groups as flows in OVS rather than Linux bridge and *iptables*, thus
increasing scalability and performance.

The native OVS firewall implementation requires kernel and user space support
for *conntrack*, thus requiring minimum versions of the Linux kernel and
Open vSwitch. All cases require Open vSwitch version 2.5 or newer.

- Kernel version 4.3 or newer includes
*conntrack*support. - Kernel version 3.3, but less than 4.3, does not include
*conntrack*support and requires building the OVS modules.

On nodes running the Open vSwitch agent, edit the

`openvswitch_agent.ini`

file and enable the firewall driver.[securitygroup] firewall_driver = openvswitch


For more information, see the developer documentation and the video.

Except where otherwise noted, this document is licensed under Creative Commons Attribution 3.0 License. See all OpenStack Legal Documents.