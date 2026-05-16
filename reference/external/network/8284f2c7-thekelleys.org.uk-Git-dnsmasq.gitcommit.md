---
title: thekelleys.org.uk Git - dnsmasq.git/commit
source: http://thekelleys.org.uk/gitweb/?p=dnsmasq.git;a=commit;h=ff325644c7afae2588583f935f4ea9b9694eb52e
kind: external
domain: network
original_date: 2016-05-03
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[thekelleys.org.uk](http://thekelleys.org.uk/gitweb/?p=dnsmasq.git;a=commit;h=ff325644c7afae2588583f935f4ea9b9694eb52e)
> 原始日期：2016-05-03
> 抓取日期：2026-05-16

# thekelleys.org.uk Git - dnsmasq.git/commit

| author | Neil Jerram <Neil.Jerram@metaswitch.com> | |
| Tue, 3 May 2016 21:45:14 +0000 (22:45 +0100) | ||
| committer | Simon Kelley <simon@thekelleys.org.uk> | |
| Tue, 3 May 2016 21:49:46 +0000 (22:49 +0100) | ||
| commit | ff325644c7afae2588583f935f4ea9b9694eb52e | |
| tree | 4dda0a5bd5330243857cdca4a881cce242576dd1 | tree | snapshot |
| parent | b97026035ecc870ea0f12f537b214237cf3d0af6 | commit | diff |

Fix for DHCP in transmission interface when --bridge-interface in use.


From f3d832b41f44c856003517c583fbd7af4dca722c Mon Sep 17 00:00:00 2001

From: Neil Jerram <Neil.Jerram@metaswitch.com>

Date: Fri, 8 Apr 2016 19:23:47 +0100

Subject: [PATCH] Fix DHCPv4 reply via --bridge-interface alias interface


Sending a DHCPv4 reply through a --bridge-interface alias interface

was inadvertently broken by


commit 65c721200023ef0023114459a8d12f8b0a24cfd8

Author: Lung-Pin Chang <changlp@cs.nctu.edu.tw>

Date: Thu Mar 19 23:22:21 2015 +0000


dhcp: set outbound interface via cmsg in unicast reply


If multiple routes to the same network exist, Linux blindly picks

the first interface (route) based on destination address, which might not be

the one we're actually offering leases. Rather than relying on this,

always set the interface for outgoing unicast DHCP packets.


because in the aliasing case, iface_index is changed from the index of

the interface on which the packet was received, to be the interface

index of the 'bridge' interface (where the DHCP context is expected to

be defined, and so needs to be looked up).


For the cmsg code that the cited commit added, we need the original

iface_index; so this commit saves that off before the aliasing code

can change it, as rcvd_iface_index, and then uses rcvd_iface_index

instead of iface_index for the cmsg code.


From f3d832b41f44c856003517c583fbd7af4dca722c Mon Sep 17 00:00:00 2001

From: Neil Jerram <Neil.Jerram@metaswitch.com>

Date: Fri, 8 Apr 2016 19:23:47 +0100

Subject: [PATCH] Fix DHCPv4 reply via --bridge-interface alias interface

Sending a DHCPv4 reply through a --bridge-interface alias interface

was inadvertently broken by

commit 65c721200023ef0023114459a8d12f8b0a24cfd8

Author: Lung-Pin Chang <changlp@cs.nctu.edu.tw>

Date: Thu Mar 19 23:22:21 2015 +0000

dhcp: set outbound interface via cmsg in unicast reply

If multiple routes to the same network exist, Linux blindly picks

the first interface (route) based on destination address, which might not be

the one we're actually offering leases. Rather than relying on this,

always set the interface for outgoing unicast DHCP packets.

because in the aliasing case, iface_index is changed from the index of

the interface on which the packet was received, to be the interface

index of the 'bridge' interface (where the DHCP context is expected to

be defined, and so needs to be looked up).

For the cmsg code that the cited commit added, we need the original

iface_index; so this commit saves that off before the aliasing code

can change it, as rcvd_iface_index, and then uses rcvd_iface_index

instead of iface_index for the cmsg code.

| src/dhcp.c | diff | blob | history |