---
title: Netfilter and the NAT of ICMP error messages
source: https://home.regit.org/2013/04/netfilter-and-icmp-error-messages/
kind: external
domain: network
author: Author Regit
original_date: 2013-04-01
fetched_at: 2026-05-16
bookmark_title: Netfilter and the NAT of ICMP error messages » To Linux and beyond !
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[home.regit.org](https://home.regit.org/2013/04/netfilter-and-icmp-error-messages/)
> 作者：Author Regit
> 原始日期：2013-04-01
> 抓取日期：2026-05-16

# Netfilter and the NAT of ICMP error messages

#### The problem

I’ve been recently working for a customer which needed consultancy because of some unexplained Netfilter behaviors related to ICMP error messages. He authorizes me to share the result of my study and I thank him for making this blog entry possible.

His problem was that one of his firewalls is using a private interconnexion with their border router and the customer did not manage to NAT all outgoing ICMP error messages.

The simplified network diagram is the following:


The DMZ is in a private network. The router has a route to the public network via the firewall and the public network address do not exists.

The firewall has set of DNAT rules to redirect a public IP to the matching private IP:

iptables -A PREROUTING -t nat -d 1.2.3.X -j DNAT --to 192.168.1.X

The interconnection between the router and firewall is made using a private network. Let’s say 192.168.42.0/24 and 192.168.42.1 for the firewall. The interface eth0 is the one used as interconnection interface.

On the firewall, some filtering rules reject some FORWARD traffic:

iptables -A FORWARD -d 192.168.1.X -j REJECT iptables -A FORWARD -d 192.168.1.Y -j REJECT

The issue is related with the ICMP unreachable messages. When someone from internet (behind the router) is sending a packet to 192.168.1.X or 192.168.1.Y then:

- If 192.168.1.X is NATed then the ICMP unreachable message is emitted and seen as coming from 1.2.3.X on eth0.
- If 192.168.1.Y is not NATed then the ICMP unreachable message is emitted and seen as coming from 192.168.42.1 on eth0.

So, a packet going to 192.168.1.Y results in a ICMP message which is not routed by the router due to the private IP.

To fix the issue, the customer has added a Source NAT rules to translate all packet coming from 192.168.42.1 to 1.2.3.1:

iptables -A POSTROUTING -t nat -p icmp -s 192.168.42.1 -o eth0 -j SNAT --to 1.2.3.1

But this rules has no effect on the ICMP unreachable message.

#### Explanations

In the case of packets going to X or Y, an ICMP message is sent. Internally the same function (called icmp_send) is used for to send the icmp error message. This is a standard function and

as such, it uses the best local source address possible. In our case the best address is 192.168.42.1 because the packet has to get back through eth0.

At current stage, there is no difference between the two ICMP packets and the result should be the same.

But if nothing is done, the packet to X will result in a packet going to the original source and containing the internal IP information: the packet has been NATed so we have 192.168.1.X and not the public IP in the original packet data contained in the ICMP message. This is a real problem as this will leak private information to the outside.

Hopefully, the packets are handled differently due to the ICMP error connection tracking module. It searches in the payload part of the ICMP error message if it belongs to existing connection. If a connection is found, the IMCP packet is marked as RELATED to the original connection. Once this is done, the ICMP nat helper makes the reverse transformation to send to the network a packet containing only public information. For packet to X, the source addresses of the ICMP messages and payload are modified to the public IP address. This explains the difference between the ICMP error message sent because of packet sent to X or sent to Y.

But this does not explain why the NAT rules inserted by the customer did not work. In fact, the response was already made: the ICMP packet is marked as belonging to a connection related to the original one. Being in a RELATED state, it will not cross the NAT in POSTROUTING as only packet with a connection in state NEW are sent to the nat tables.

The validation of this study can be done by using marking and logging. If we log a packet which belong to a RELATED connection and if we are sure that the original connection is the one we are tracing then our hypothesis is validated. Getting a RELATED connection is easy with the filter: “-m conntrack –ctstate RELATED”. To prove that the packet is RELATED to the original connection, we have to use the fact that RELATED connection inherit of the connection mark of the originating connection. Thus, if we set a connection mark with the CONNMARK target, we will be able to match it in the ICMP error message. The following rules implement this:

iptables -t mangle -A PREROUTING -d 1.2.3.4 -j CONNMARK --set-mark 1 iptables -A OUTPUT -t mangle -m state --state RELATED -m connmark --mark 1 -j LOG

And it logs an ICMP error message when we try to reach 1.2.3.4.

#### Other debug methods

##### Using conntrack

The conntrack utils can be used to display connection tracking events by using the -E flag:

# conntrack -E [NEW] tcp 6 120 SYN_SENT src=192.168.1.12 dst=91.121.96.152 sport=53398 dport=22 [UNREPLIED] src=91.121.96.152 dst=192.168.1.12 sport=22 dport=53398 [UPDATE] tcp 6 60 SYN_RECV src=192.168.1.12 dst=91.121.96.152 sport=53398 dport=22 src=91.121.96.152 dst=192.168.1.12 sport=22 dport=53398 [UPDATE] tcp 6 432000 ESTABLISHED src=192.168.1.12 dst=91.121.96.152 sport=53398 dport=22 src=91.121.96.152 dst=192.168.1.12 sport=22 dport=53398 [ASSURED]

This can be really useful to see what transformation are made by the connection tracking system. But this does not work in your case because the icmp message does not trigger any connection creation and so no event.

##### Using TRACE target

The TRACE target is a really useful tool. It allows you to see which rules are matched by a packet. It’s usage is really simple. For example, if we want to trace all ICMP traffic coming to the box:

iptables -A PREROUTING -t raw -p icmp -j TRACE

In our test system, the result was the following:

[ 5281.733217] TRACE: raw:PREROUTING:policy:2 IN=eth0 OUT= MAC=08:00:27:a9:f5:30:0a:00:27:00:00:00:08:00 SRC=192.168.56.1 DST=1.2.3.4 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=0 DF PROTO=ICMP TYPE=8 CODE=0 ID=12114 SEQ=1 [ 5281.737057] TRACE: nat:PREROUTING:rule:1 IN=eth0 OUT= MAC=08:00:27:a9:f5:30:0a:00:27:00:00:00:08:00 SRC=192.168.56.1 DST=1.2.3.4 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=0 DF PROTO=ICMP TYPE=8 CODE=0 ID=12114 SEQ=1 [ 5281.737057] TRACE: nat:PREROUTING:rule:2 IN=eth0 OUT= MAC=08:00:27:a9:f5:30:0a:00:27:00:00:00:08:00 SRC=192.168.56.1 DST=1.2.3.4 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=0 DF PROTO=ICMP TYPE=8 CODE=0 ID=12114 SEQ=1 [ 5281.737057] TRACE: filter:FORWARD:rule:1 IN=eth0 OUT=eth1 MAC=08:00:27:a9:f5:30:0a:00:27:00:00:00:08:00 SRC=192.168.56.1 DST=192.168.42.4 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=ICMP TYPE=8 CODE=0 ID=12114 SEQ=1

In the raw TABLE, in PREROUTING the policy is applied (here ACCEPT). In nat PREROUTING the first rule is matching (a mark rule) and the second one is matching too. Finally in FORWARD, the first rule is matching (here the REJECT rule). TRACE is only following the initial packet and thus does not display any information about the ICMP error message.

#### Conclusion

So Netfilter’s behavior is correct when it translate back the elements initially transformed by NAT. The surprising part comes from the fact that the NAT rules in POSTROUTING are not reach. But this is needed to avoid any complicated issue by doing multiple transformation. Regarding interconnexion with router, you should really use a public network if you want your ICMP error messages to be seen on Internet.