---
title: Bug #1616268 “Stale namespace removal causing “RTNETLINK answers...” : Bugs : kolla
source: https://bugs.launchpad.net/kolla/+bug/1616268
kind: external
domain: network
original_date: 2016-08-24
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[bugs.launchpad.net](https://bugs.launchpad.net/kolla/+bug/1616268)
> 原始日期：2016-08-24
> 抓取日期：2026-05-16

# Bug #1616268 “Stale namespace removal causing “RTNETLINK answers...” : Bugs : kolla

# Stale namespace removal causing "RTNETLINK answers: Invalid argument" errors

| Affects | Status | Importance | Assigned to | Milestone | |
|---|---|---|---|---|---|
| kolla |
Fix Released
|
Critical
|
Jeffrey Zhang | ||
| Liberty |
Won't Fix
|
Critical
|
Unassigned | ||
| Mitaka |
Fix Released
|
Critical
|
Jeffrey Zhang | ||
| Newton |
Fix Released
|
Critical
|
Jeffrey Zhang | ||
| Ocata |
Fix Released
|
Critical
|
Jeffrey Zhang |

### Bug Description

etherpad discuss: https:/

I've been able to replicate this by creating routers, and assigning a gateway, which in turn, creates a 'qrouter' and 'snat' namespace with the same uuid (expected).

Upon clearing the gateway and removing the router, we're left with our network nodes, and compute nodes (due to DVR) nodes having stale namespaces which causes excess logging errors, and breaks networking to a certain extent.

Running 'ip netns show' displays said "RTNETLINK answers: Invalid argument" errors.

Restarting neutron and openvswitch docker containers does not fix the issue. I tried removing invalid namespaces by stopping one container at a time, and running "ip netns delete $namespace" without any success.

The only way(s) that I have found which works, is stop all neutron and openvswitch containers, then you can run "ip netns delete $namespace", or reboot.

Example to reproduce:

for y in {1..25}; do neutron router-create MT-TEST-${y}; neutron router-gateway-set MT-TEST-${y} External; done

for y in {1..25}; do neutron router-

Note: "External" being the name of the external network, naturally this value should change based upon the environment being tested.

And to clear (per host):

docker ps -a | egrep '(neutron|

for y in $(ls /var/run/netns/); do ip netns delete $y; done

docker ps -a | egrep '(neutron|

Or reboot (naturally this is a last resort).

I have also tried setting "router_

Current version of Neutron is:

(neutron-

openstack-

(neutron-

Name : openstack-neutron

Epoch : 1

Version : 8.1.2

Release : 1.el7

Architecture: noarch

Install Date: Thu Aug 4 11:33:24 2016

Group : Unspecified

Size : 75210

License : ASL 2.0

Signature : RSA/SHA1, Tue Jun 21 08:24:35 2016, Key ID f9b9fee7764429e6

Source RPM : openstack-

Build Date : Thu Jun 16 20:53:15 2016

Build Host : c1bk.rdu2.

Relocations : (not relocatable)

Packager : CBS <email address hidden>

Vendor : CentOS

URL : http://

Summary : OpenStack Networking Service

Description :

Neutron is a virtual network service for Openstack. Just like

OpenStack Nova provides an API to dynamically request and configure

virtual servers, Neutron provides an API to dynamically request and

configure virtual networks. These networks connect "interfaces" from

other OpenStack services (e.g., virtual NICs from Nova VMs). The

Neutron API supports extensions to provide advanced network

capabilities (e.g., QoS, ACLs, network monitoring, etc.)

Current version of openvswitch is:

(openvswitch-

openvswitch-

(openvswitch-

Name : openvswitch

Version : 2.5.0

Release : 2.el7

Architecture: x86_64

Install Date: Fri Jul 8 05:23:26 2016

Group : Unspecified

Size : 10191793

License : ASL 2.0 and LGPLv2+ and SISSL

Signature : RSA/SHA1, Thu Jun 2 22:51:51 2016, Key ID f9b9fee7764429e6

Source RPM : openvswitch-

Build Date : Fri Mar 18 15:00:30 2016

Build Host : c1bk.rdu2.

Relocations : (not relocatable)

Packager : CBS <email address hidden>

Vendor : CentOS

URL : http://

Summary : Open vSwitch daemon/

Description :

Open vSwitch provides standard network bridging functions and

support for the OpenFlow protocol for remote per-flow control of

traffic.

I will attach a few more logs shortly.

| Changed in kolla: | |
importance:
|
Undecided → Critical |
status:
|
New → Confirmed |
milestone:
|
none → 3.0.0 |
milestone:
|
3.0.0 → newton-3 |

| Changed in kolla: | |
milestone:
|
newton-3 → newton-rc1 |

| Changed in kolla: | |
milestone:
|
newton-rc1 → newton-rc2 |

| Changed in kolla: | |
assignee:
|
nobody → Jeffrey Zhang (jeffrey4l) |

| Changed in kolla: | |
milestone:
|
newton-rc2 → newton-rc3 |

description:
|
updated |