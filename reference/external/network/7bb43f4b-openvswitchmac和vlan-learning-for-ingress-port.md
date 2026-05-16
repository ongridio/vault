---
title: openvswitch——mac和vlan learning for ingress port
source: http://www.cnblogs.com/CasonChan/p/4754486.html
kind: external
domain: network
author: CasonChan
original_date: 2015-08-24
fetched_at: 2026-05-16
bookmark_title: openvswitch——mac和vlan learning for ingress port - CasonChan - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/CasonChan/p/4754486.html)
> 作者：CasonChan
> 原始日期：2015-08-24
> 抓取日期：2026-05-16

# openvswitch——mac和vlan learning for ingress port

## openvswitch——mac和vlan learning for ingress port

对于普通的switch，都会有这个学习的过程，当一个包到来的时候，由于包里面有MAC，VLAN Tag，以及从哪个口进来的这个信息。于是switch学习后，维护了一个表格port –> MAC –> VLAN Tag。

这样以后如果有需要发给这个MAC的包，不用ARP，switch自然之道应该发给哪个port，应该打什么VLAN Tag。

OVS也要学习这个，并维护三个之间的mapping关系。

在我们的例子中，无论是从port进来的本身就带Tag的，还是从port 2, 3, 4进来的后来被打上Tag的，都需要学习。

sudo ovs-ofctl add-flow helloworld "table=2 actions=learn(table=10, NXM_OF_VLAN_TCI[0..11], NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[], load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]), resubmit(,3)"

这一句比较难理解。

learn表示这是一个学习的action

table 10，这是一个MAC learning table，学习的结果会放在这个table中。

NXM_OF_VLAN_TCI这个是VLAN Tag，在MAC Learning table中，每一个entry都是仅仅对某一个VLAN来说的，不同VLAN的learning table是分开的。在学习的结果的entry中，会标出这个entry是对于哪个VLAN的。

NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[]这个的意思是当前包里面的MAC Source Address会被放在学习结果的entry里面的dl_dst里面。这是因为每个switch都是通过Ingress包来学习，某个MAC从某个 port进来，switch就应该记住以后发往这个MAC的包要从这个port出去，因而MAC source address就被放在了Mac destination address里面，因为这是为发送用的。

NXM_OF_IN_PORT[]->NXM_NX_REG0将portf放入register.

一般对于学习的entry还需要有hard_timeout，这是的每个学习结果都会expire，需要重新学习。

我们再来分析一个实践中，openstack中使用openvswitch的情况，这是br-tun上的规则。

**cookie=0x0, duration=802188.071s, table=10, n_packets=4885, n_bytes=347789, idle_age=730, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1 **cookie=0x0, duration=802187.786s, table=20,
n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0
actions=resubmit(,21)

**cookie=0x0, duration=802038.514s, table=20, n_packets=1239, n_bytes=83620, idle_age=735, hard_age=65534, priority=2,dl_vlan=1,dl_dst=fa:16:3e:7e:ab:cc actions=strip_vlan,set_tunnel:0x3e9,output:2**

cookie=0x0, duration=802187.653s, table=21, n_packets=17, n_bytes=1426, idle_age=65534, hard_age=65534, priority=0 actions=drop

cookie=0x0, duration=802055.878s, table=21, n_packets=40, n_bytes=1736, idle_age=65534, hard_age=65534, dl_vlan=1 actions=strip_vlan,set_tunnel:0x3e9,output:2

这里table 10是用来学习的。table 20是learning table。如果table 20是空的，也即还没有学到什么，则会通过priority=0的规则resubmit到table 21.

table 21是发送规则，将br-int上的vlan tag消除，然后打上gre tunnel的id。

上面的情况中，table 20不是空的，也即发送给dl_dst=fa:16:3e:7e:ab:cc的包不用走默认规则，直接通过table 20就发送出去了。

table 20的规则是通过table 10学习得到的，table 10是一个接受规则。最终output 1，发送给了br-int

NXM_OF_VLAN_TCI[0..11]是记录vlan tag，所以学习结果中有dl_vlan=1

NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[]是将mac source address记录，所以结果中有dl_dst=fa:16:3e:7e:ab:cc

load:0->NXM_OF_VLAN_TCI[]意思是发送出去的时候，vlan tag设为0，所以结果中有actions=strip_vlan

load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[]意思是发出去的时候，设置tunnul id，所以结果中有set_tunnel:0x3e9

output:NXM_OF_IN_PORT[]意思是发送给哪个port，由于是从port2进来的，因而结果中有output:2

测试一：从port 1来一个vlan为20的mac为50:00:00:00:00:01的包

$ sudo ovs-appctl ofproto/trace helloworld in_port=1,vlan_tci=20,dl_src=50:00:00:00:00:01 -generate

Flow: metadata=0,in_port=1,vlan_tci=0x0014,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00,dl_type=0x0000

Rule: table=0 cookie=0 priority=0

OpenFlow actions=resubmit(,1)

Resubmitted flow: unchanged

Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0

Resubmitted odp: drop

Rule: table=1 cookie=0 priority=99,in_port=1

OpenFlow actions=resubmit(,2)

Resubmitted flow: unchanged

Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0

Resubmitted odp: drop

Rule: table=2 cookie=0

OpenFlow
actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)

Resubmitted flow: unchanged

Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0

Resubmitted odp: drop

No match

Final flow: unchanged

Relevant fields:
skb_priority=0,in_port=1,vlan_tci=0x0014/0x0fff,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000,nw_frag=no

Datapath actions: drop

$ sudo ovs-ofctl dump-flows helloworld

NXST_FLOW reply (xid=0x4):

cookie=0x0, duration=90537.25s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=resubmit(,1)

cookie=0x0, duration=90727.209s, table=0, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534,
dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop

cookie=0x0, duration=90662.724s, table=0, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534,
dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop

cookie=0x0, duration=86147.941s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=2,vlan_tci=0x0000
actions=mod_vlan_vid:20,resubmit(,2)

cookie=0x0, duration=86147.941s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=4,vlan_tci=0x0000
actions=mod_vlan_vid:30,resubmit(,2)

cookie=0x0, duration=86147.941s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=3,vlan_tci=0x0000
actions=mod_vlan_vid:30,resubmit(,2)

cookie=0x0, duration=86278.986s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=1
actions=resubmit(,2)

cookie=0x0, duration=86357.407s, table=1, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop

cookie=0x0, duration=83587.281s, table=2, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534,
actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)

**cookie=0x0, duration=31.258s, table=10, n_packets=0,
n_bytes=0, idle_age=31, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01
actions=load:0x1->NXM_NX_REG0[0..15]**

table 10多了一条，vlan为20，dl_dst为50:00:00:00:00:01，发送的时候从port 1出去。

测试二：从port 2进来，被打上了vlan 20，mac为50:00:00:00:00:02

$ sudo ovs-appctl ofproto/trace helloworld in_port=2,dl_src=50:00:00:00:00:02 -generate

Flow: metadata=0,in_port=2,vlan_tci=0x0000,dl_src=50:00:00:00:00:02,dl_dst=00:00:00:00:00:00,dl_type=0x0000

Rule: table=0 cookie=0 priority=0

OpenFlow actions=resubmit(,1)

Resubmitted flow: unchanged

Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0

Resubmitted odp: drop

Rule: table=1 cookie=0 priority=99,in_port=2,vlan_tci=0x0000

OpenFlow actions=mod_vlan_vid:20,resubmit(,2)

Resubmitted flow:
metadata=0,in_port=2,dl_vlan=20,dl_vlan_pcp=0,dl_src=50:00:00:00:00:02,dl_dst=00:00:00:00:00:00,dl_type=0x0000

Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0

Resubmitted odp: drop

Rule: table=2 cookie=0

OpenFlow
actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)

Resubmitted flow: unchanged

Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0

Resubmitted odp: drop

No match

Final flow: unchanged

Relevant fields:
skb_priority=0,in_port=2,vlan_tci=0x0000,dl_src=50:00:00:00:00:02,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000,nw_frag=no

Datapath actions: drop

$ sudo ovs-ofctl dump-flows helloworld

NXST_FLOW reply (xid=0x4):

cookie=0x0, duration=90823.14s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=resubmit(,1)

cookie=0x0, duration=91013.099s, table=0, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534,
dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop

cookie=0x0, duration=90948.614s, table=0, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534,
dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop

cookie=0x0, duration=86433.831s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=2,vlan_tci=0x0000
actions=mod_vlan_vid:20,resubmit(,2)

cookie=0x0, duration=86433.831s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=4,vlan_tci=0x0000
actions=mod_vlan_vid:30,resubmit(,2)

cookie=0x0, duration=86433.831s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=3,vlan_tci=0x0000
actions=mod_vlan_vid:30,resubmit(,2)

cookie=0x0, duration=86564.876s, table=1, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534, priority=99,in_port=1
actions=resubmit(,2)

cookie=0x0, duration=86643.297s, table=1, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop

cookie=0x0, duration=83873.171s, table=2, n_packets=0, n_bytes=0,
idle_age=65534, hard_age=65534,
actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)

**cookie=0x0, duration=4.472s, table=10, n_packets=0,
n_bytes=0, idle_age=4, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:02
actions=load:0x2->NXM_NX_REG0[0..15]**

cookie=0x0, duration=317.148s, table=10, n_packets=0, n_bytes=0,
idle_age=317, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01
actions=load:0x1->NXM_NX_REG0[0..15]

*摘录自http://www.cnblogs.com/popsuper1982/p/3800535.html*

-------------------------

No pains, no gains