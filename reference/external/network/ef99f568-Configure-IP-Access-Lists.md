---
title: Configure IP Access Lists
source: https://www.cisco.com/c/en/us/support/docs/security/ios-firewall/23602-confaccesslists.html
kind: external
domain: network
author: Cisco Engineers Cisco TAC Engineers
original_date: 2025-10-15
fetched_at: 2026-05-16
bookmark_title: Configuring IP Access Lists - Cisco
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cisco.com](https://www.cisco.com/c/en/us/support/docs/security/ios-firewall/23602-confaccesslists.html)
> 作者：Cisco Engineers Cisco TAC Engineers
> 原始日期：2025-10-15
> 抓取日期：2026-05-16

# Configure IP Access Lists

Updated:October 15, 2025

Document ID:23602

The documentation set for this product strives to use bias-free language. For the purposes of this documentation set, bias-free is defined as language that does not imply discrimination based on age, disability, gender, racial identity, ethnic identity, sexual orientation, socioeconomic status, and intersectionality. Exceptions may be present in the documentation due to language that is hardcoded in the user interfaces of the product software, language used based on RFP documentation, or language that is used by a referenced third-party product. Learn more about how Cisco is using Inclusive Language.

This document describes various types of IP Access Control Lists (ACLs) and how they can filter network traffic.

There are no specific prerequisites for this document.

This document discusses various types of ACLs. Some of these are present since Cisco IOS®Software Releases 8.3 and others were introduced in later software releases. This is noted in the discussion of each type.

The information in this document was created from the devices in a specific lab environment. All of the devices used in this document started with a cleared (default) configuration. If your network is live, ensure that you understand the potential impact of any command.

Refer toCisco Technical Tips Conventionsfor more information on document conventions.

This document describes how IP access control lists (ACLs) can filter network traffic. It also contains brief descriptions of the IP ACL types, feature availability, and an example of use in a network.

**Note**: RFC 1700 contains assigned numbers of well-known ports.RFC 1918 contains address allocation for private Internets, IP addresses which must not normally be seen on the Internet.

**Note**: Only registered Cisco users can access internal information.

**Note**: ACLs can also be used to define traffic to Network Address Translate (NAT), encrypt, or filter non-IP protocols such as AppleTalk or IPX. A discussion of these functions is outside the scope of this document.

Masks are used in IP access control lists (ACLs) to define which parts of an IP address must match and which parts can vary. When configuring IP addresses on interfaces, you use a subnet mask, which starts with larger values (255s) on the left, for example, the IP address 10.1.1.129 with subnet mask 255.255.255.0. In contrast, ACLs use a wildcard mask (also called an inverse mask), which works in the opposite way. For example, the wildcard mask 0.0.0.255 means that the first three octets must match exactly, while the last octet can vary. In binary terms, each 0 in the wildcard mask means, this bit must match exactly, and each 1 means, this bit can be ignored (do not care). The next table illustrates this concept in more detail.

| Mask Example | |
|---|---|
| network address (traffic that is to be processed) | 10.1.1.0 |
| subnet mask | 255.255.255.0 |
| inverse mask | 0.0.0.255 |
| network address (binary) | 00001010.00000001.00000001.00000000 |
| inverse mask (binary) | 00000000.00000000.00000000.11111111 |

Based on the binary inverse mask, you can see that the first three octets must match the given binary network address exactly (00001010.00000001.00000001). The last set of numbers are do not cares (.11111111). Therefore, all traffic that begins with 10.1.1. matches since the last octet is do not care. Consequently, network addresses 10.1.1.1 through 10.1.1.255 (10.1.1.x) are processed.

In order to determine the ACL inverse mask, you need to subtract the subnet mask from 255.255.255.255 In this example, the inverse mask is determined for network address 172.16.1.0 with a subnet mask of 255.255.255.0.

-
255.255.255.255 - 255.255.255.0 (subnet mask) = 0.0.0.255 (inverse mask)


Notice the ACL equivalents.

-
The source/wildcard of 0.0.0.0/255.255.255.255 means any.

-
The source/wildcard of 10.1.1.2/0.0.0.0 is the same as host 10.1.1.2.


**Note**: Subnet masks can also be represented as a fixed length notation. For example, 192.168.10.0/24 represents 192.168.10.0 255.255.255.0.

This list describes how to summarize a range of networks into a single network for ACL optimization. Consider these networks.

192.168.32.0/24 192.168.33.0/24 192.168.34.0/24 192.168.35.0/24 192.168.36.0/24 192.168.37.0/24 192.168.38.0/24 192.168.39.0/24

The first two octets and the last octet are the same for each network. This table is an explanation of how to summarize these into a single network.

The third octet for the previous networks can be written as seen in this table, correspondent to the octet bit position and address value for each bit.

Decimal |
128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
| 32 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 |
| 33 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 1 |
| 34 | 0 | 0 | 1 | 0 | 0 | 0 | 1 | 0 |
| 35 | 0 | 0 | 1 | 0 | 0 | 0 | 1 | 1 |
| 36 | 0 | 0 | 1 | 0 | 0 | 1 | 0 | 0 |
| 37 | 0 | 0 | 1 | 0 | 0 | 1 | 0 | 1 |
| 38 | 0 | 0 | 1 | 0 | 0 | 1 | 1 | 0 |
| 39 | 0 | 0 | 1 | 0 | 0 | 1 | 1 | 1 |
| M | M | M | M | M | D | D | D |

Since the first five bits match, the previous eight networks can be summarized into one network (192.168.32.0/21 or 192.168.32.0 255.255.248.0). All eight possible combinations of the three low-order bits are relevant for the network ranges in question. This command defines an ACL that permits this network. If you subtract 255.255.248.0 (subnet mask) from 255.255.255.255, it yields 0.0.7.255.

access-list acl_permit permit ip 192.168.32.0`0.0.7.255`


Consider this set of networks for further explanation.

192.168.146.0/24 192.168.147.0/24 192.168.148.0/24 192.168.149.0/24

The first two octets and the last octet are the same for each network. This table is an explanation of how to summarize these.

The third octet for the previous networks can be written as seen in this table, correspondent to the octet bit position and address value for each bit.

Decimal |
128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
| 146 | 1 | 0 | 0 | 1 | 0 | 0 | 1 | 0 |
| 147 | 1 | 0 | 0 | 1 | 0 | 0 | 1 | 1 |
| 148 | 1 | 0 | 0 | 1 | 0 | 1 | 0 | 0 |
| 149 | 1 | 0 | 0 | 1 | 0 | 1 | 0 | 1 |
| M | M | M | M | M | ? | ? | ? |

Unlike the previous example, you cannot summarize these networks into a single network. If they are summarized to a single network, they become 192.168.144.0/21 because there are five bits similar in the third octet. This summarized network 192.168.144.0/21 covers a range of networks from 192.168.144.0 to 192.168.151.0. Among these, 192.168.144.0, 192.168.145.0, 192.168.150.0, and 192.168.151.0 networks are not in the given list of four networks. In order to cover the specific networks in question, you need a minimum of two summarized networks. The given four networks can be summarized into these two networks:

-
For networks 192.168.146.x and 192.168.147.x, all bits match except for the last one, which is a do not care. This can be written as 192.168.146.0/23 (or 192.168.146.0 255.255.254.0).

-
For networks 192.168.148.x and 192.168.149.x, all bits match except for the last one, which is a do not care. This can be written as 192.168.148.0/23 (or 192.168.148.0 255.255.254.0).


This output defines a summarized ACL for the previously networks.


!--- This command is used to allow access access for devices with IP

!--- addresses in the range from 192.168.146.0 to 192.168.147.254.access-list 10 permit 192.168.146.0`0.0.1.255`



!--- This command is used to allow access access for devices with IP

!--- addresses in the range from 192.168.148.0 to 192.168.149.254access-list 10 permit 192.168.148.0`0.0.1.255`


Traffic that comes into the router is compared to ACL entries based on the order that the entries occur in the router. New statements are added to the end of the list. The router continues to look until it has a match. If no matches are found when the router reaches the end of the list, the traffic is denied. For this reason, you must have the frequently hit entries at the top of the list. Every ACL has an implicit deny at the end for any traffic that is not explicitly permitted.. A single-entry ACL with only one deny entry can deny all traffic. You must have at least one permit statement in an ACL or all traffic is blocked. These two ACLs (101 and 102) have the same effect.


!--- This command is used to permit IP traffic from 10.1.1.0

!--- network to 172.16.1.0 network. All packets with a source

!--- address not in this range will be rejected.access-list 101 permit ip 10.1.1.0 0.0.0.255 172.16.1.0`0.0.0.255`



!--- This command is used to permit IP traffic from 10.1.1.0

!--- network to 172.16.1.0 network. All packets with a source

!--- address not in this range will be rejected.access-list 102 permit ip 10.1.1.0 0.0.0.255 172.16.1.0`0.0.0.255`


access-list 102 deny ip any any

In the next example, the last entry is sufficient. You do not need the first three entries because IP includes TCP, User Datagram Protocol (UDP), and Internet Control Message Protocol (ICMP).

!--- This command is used to permit Telnet traffic

!--- from machine 10.1.1.2 to machine 172.16.1.1.access-list 101 permit tcp host 10.1.1.2 host 172.16.1.1 eq telnet

!--- This command is used to permit tcp traffic from

!--- 10.1.1.2 host machine to 172.16.1.1 host machine.access-list 101 permit tcp host 10.1.1.2 host 172.16.1.1

!--- This command is used to permit udp traffic from

!--- 10.1.1.2 host machine to 172.16.1.1 host machine.access-list 101 permit udp host 10.1.1.2 host 172.16.1.1

!--- This command is used to permit ip traffic from

!--- 10.1.1.0 network to 172.16.1.10 network.access-list 101 permit ip 10.1.1.0 0.0.0.255 172.16.1.0`0.0.0.255`


Not only can you define ACL source and destination, but you can also define ports, ICMP message types, and other parameters. A good source of information for well-known ports is RFC 1700 . ICMP message types are explained in RFC 792 .

The router can display descriptive text on some of the well-known ports. Select a**?**for help.

access-list 102 permit tcp host 10.1.1.1 host 172.16.1.1 eq ?bgp Border Gateway Protocol (179) chargen Character generator (19) cmd Remote commands (rcmd, 514)

During configuration, the router also converts numeric values to more user-friendly values. This is an example where you type the ICMP message type number, and it causes the router to convert the number to a name.

access-list 102 permit icmp host 10.1.1.1 host 172.16.1.1 14

becomes

access-list 102 permit icmp host 10.1.1.1 host 172.16.1.1 timestamp-reply

You can define ACLs and still not apply them. But, the ACLs have no effect until they are applied to the interface of the router. It is a good practice to apply the ACL on the interface closest to the source of the traffic. As shown in this example, when you try to block traffic from source to destination, you can apply an inbound ACL to E0 on router A instead of an outbound list to E1 on router C. An access-list has adeny ip any implicitly at the end of any access-list. If traffic is related to a DHCP request and if it is not explicitly permitted, the traffic is dropped because when you look at DHCP request in IP, the source address is s=0.0.0.0 (Ethernet1/0), d=255.255.255.255, len 604, rcvd 2 UDP src=68, dst=67. Notice that the source IP address is 0.0.0.0 and destination address is 255.255.255.255. Source port is 68 and destination 67. Therefore, you must permit this kind of traffic in your access-list or else the traffic is dropped due to implicit deny at the end of the statement.

**Note**: For UDP traffic to pass through, UDP traffic must also be permitted explicitly by the ACL.

The router uses the terms in, out, source, and destination as references. Traffic on the router can be compared to traffic on the highway. If you were a law enforcement officer in Pennsylvania and wanted to stop a truck that travels from Maryland to New York, the source of the truck is Maryland, and the destination of the truck is New York. The roadblock could be applied at the Pennsylvania–New York border (out) or the Maryland–Pennsylvania border (in).

When you refer to a router, these terms have these meanings.

-
Out—Traffic that has already been through the router and leaves the interface. The source is where it has been, on the other side of the router, and the destination is where it goes.

-
In—Traffic that arrives on the interface and then goes through the router. The source is where it has been and the destination is where it goes, on the other side of the router.

-
Inbound—If the access list is inbound, when the router receives a packet, the Cisco IOS software checks the criteria statements of the access list for a match. If the packet is permitted, the software continues to process the packet. If the packet is denied, the software discards the packet.

-
Outbound—If the access list is outbound, after the software receives and routes a packet to the outbound interface, the software checks the criteria statements of the access list for a match. If the packet is permitted, the software transmits the packet. If the packet is denied, the software discards the packet.


The in ACL has a source on a segment of the interface to which it is applied and a destination off of any other interface. The out ACL has a source on a segment of any interface other than the interface to which it is applied and a destination off of the interface to which it is applied.

When you edit an ACL, it requires special attention. For example, if you intend to delete a specific line from a numbered ACL that exists as shown here, the entire ACL is deleted.

!--- The access-list 101 denies icmp from any to any networkrouter#

!--- but permits IP traffic from any to any network.configure terminalEnter configuration commands, one per line. End with CNTL/Z. router(config)#access-list 101 deny icmp any anyrouter(config)#access-list 101 permit ip any anyrouter(config)#^Zrouter#show access-listExtended IP access list 101 deny icmp any any permit ip any any router# *Mar 9 00:43:12.784: %SYS-5-CONFIG_I: Configured from console by console router#configure terminalEnter configuration commands, one per line. End with CNTL/Z. router(config)#no access-list 101 deny icmp any anyrouter(config)#^Zrouter#show access-listrouter# *Mar 9 00:43:29.832: %SYS-5-CONFIG_I: Configured from console by console

Copy the configuration of the router to a TFTP server or a text editor such as Notepad in order to edit numbered ACLs. Then make any changes and copy the configuration back to the router.

You can also do this.

`router#`

configure terminalEnter configuration commands, one per line. router(config)#ip access-list extended test!--- Permits IP traffic from 10.2.2.2 host machine to 10.3.3.3 host machine.router(config-ext-nacl)#permit ip host 10.2.2.2 host 10.3.3.3!--- Permits www traffic from 10.1.1.1 host machine to 10.5.5.5 host machine.router(config-ext-nacl)#permit tcp host 10.1.1.1 host 10.5.5.5 eq www!--- Permits icmp traffic from any to any network.router(config-ext-nacl)#permit icmp any any!--- Permits dns traffic from 10.6.6.6 host machine to 10.10.10.0 network.router(config-ext-nacl)#permit udp host 10.6.6.6 10.10.10.0 0.0.0.255 eq domainrouter(config-ext-nacl)#^Z 1d00h: %SYS-5-CONFIG_I: Configured from console by consoles-l router#show access-listExtended IP access list test permit ip host 10.2.2.2 host 10.3.3.3 permit tcp host 10.1.1.1 host 10.5.5.5 eq www permit icmp any any permit udp host 10.6.6.6 10.10.10.0 0.0.0.255 eq domain

Any deletions are removed from the ACL and any additions are made to the end of the ACL.

router#configure terminalEnter configuration commands, one per line. End with CNTL/Z. router(config)#ip access-list extended test!--- ACL entry deleted.router(config-ext-nacl)#no permit icmp any any!--- ACL entry added.router(config-ext-nacl)#permit gre host 10.4.4.4 host 10.8.8.8router(config-ext-nacl)#^Z1d00h: %SYS-5-CONFIG_I: Configured from console by consoles-l router#show access-listExtended IP access list test permit ip host 10.2.2.2 host 10.3.3.3 permit tcp host 10.1.1.1 host 10.5.5.5 eq www permit udp host 10.6.6.6 10.10.10.0 0.0.0.255 eq domain permit gre host 10.4.4.4 host 10.8.8.8

You can also add ACL lines to numbered standard or numbered extended ACLs by sequence number in Cisco IOS. This is a sample of the configuration:

Configure the extended ACL in this way:

Router(config)#access-list 101 permit tcp any anyRouter(config)#access-list 101 permit udp any anyRouter(config)#access-list 101 permit icmp any anyRouter(config)#exitRouter#

Issue the**show access-list**command in order to view the ACL entries. The sequence numbers such as 10, 20, and 30 also appear here.

Router#show access-listExtended IP access list 101 10 permit tcp any any 20 permit udp any any 30 permit icmp any any

Add the entry for the access list 101 with the sequence number 5.

Example 1:

Router#configure terminalEnter configuration commands, one per line. End with CNTL/Z. Router(config)#ip access-list extended 101Router(config-ext-nacl)#5 deny tcp any any eq telnetRouter(config-ext-nacl)#exitRouter(config)#exitRouter#

In the**show access-list**command output, the sequence number 5 ACL is added as the first entry to the access-list 101.

Router#show access-listExtended IP access list 1015 deny tcp any any eq telnet10 permit tcp any any 20 permit udp any any 30 permit icmp any any Router#

Example 2:

internetrouter#show access-listsExtended IP access list 101 10 permit tcp any any 15 permit tcp any host 172.16.2.9 20 permit udp host 172.16.1.21 any 30 permit udp host 172.16.1.22 any internetrouter#configure terminalEnter configuration commands, one per line. End with CNTL/Z. internetrouter(config)#ip access-list extended 101internetrouter(config-ext-nacl)#18 per tcp any host 172.16.2.11internetrouter(config-ext-nacl)#^Zinternetrouter#show access-listsExtended IP access list 101 10 permit tcp any any 15 permit tcp any host 172.16.2.9 18 permit tcp any host 172.16.2.11 20 permit udp host 172.16.1.21 any 30 permit udp host 172.16.1.22 any internetrouter#

Similarly, you can configure the standard access list in this way:

internetrouter(config)#access-list 2 permit 172.16.1.2internetrouter(config)#access-list 2 permit 172.16.1.10internetrouter(config)#access-list 2 permit 172.16.1.11internetrouter#show access-listsStandard IP access list 2 30 permit 172.16.1.11 20 permit 172.16.1.10 10 permit 172.16.1.2 internetrouter(config)#ip access-list standard 2internetrouter(config-std-nacl)#25 per 172.16.1.7internetrouter(config-std-nacl)#15 per 172.16.1.16internetrouter#show access-listsStandard IP access list 215 permit 172.16.1.1630 permit 172.16.1.11 20 permit 172.16.1.1025 permit 172.16.1.710 permit 172.16.1.2

The major difference in a standard access list is that the Cisco IOS adds an entry in descendant order of the IP address, not on a sequence number.

This example shows the different entries, for example, how to permit an IP address (192.168.100.0) or the networks (10.10.10.0).

internetrouter#show access-listsStandard IP access list 19 10 permit 192.168.100.0 15 permit 10.10.10.0, wildcard bits 0.0.0.255 19 permit 10.101.110.0, wildcard bits 0.0.0.255 25 deny any

Add the entry in access list 2 in order to permit the IP Address 172.22.1.1:

internetrouter(config)#ip access-list standard 2internetrouter(config-std-nacl)#18 permit 172.22.1.1

This entry is added in the top of the list in order to give priority to the specific IP address rather than network.

internetrouter#show access-listsStandard IP access list 19 10 permit 192.168.100.018 permit 172.22.1.115 permit 10.10.10.0, wildcard bits 0.0.0.255 19 permit 10.101.110.0, wildcard bits 0.0.0.255 25 deny any

**Note**: The previous ACLs are not supported in Security Appliance such as the ASA/PIX Firewall.

Guidelines to change access-lists when they are applied to crypto maps.

-
If you add to a current access-list configuration, there is no need to remove the crypto map. If you add to them directly without the removal of the crypto map, then that is supported and acceptable.

-
If you need to modify or delete access-list entry from a current access-lists, then you must remove the crypto map from the interface. After you remove crypto map, make all changes to the access-list and re-add the crypto map. If you make changes such as the deletion of the access-list without the removal of the crypto map, this is not supported and can result in unpredictable behavior.


How do I remove an ACL from an interface?

Go into configuration mode and enter ** no ** in front of the ** access-group ** command, as shown in this example, in order to remove an ACL from an interface.

interface <interface-name> no ip access-group <acl> {in|out}

What do I do when too much traffic is denied?

If too much traffic is denied, study the logic of your list or try to define and apply an additional broader list. The ** show ip access-lists ** command provides a packet count that shows which ACL entry is hit. The log keyword at the end of the individual ACL entries shows the ACL number and whether the packet was permitted or denied, in addition to port-specific information.

**Note**: The log-input keyword exists in Cisco IOS Software Release 11.2 and later. Use of this keyword logs additional information about traffic that matches the ACL, such as the ingress interface and source MAC address where applicable.

How do I debug at the packet level that uses a Cisco router?

This procedure explains the debug process. Before you begin, be certain that there are no currently applied ACLs, that there is an ACL, and that fast switching is not disabled.

**Note**: Use extreme caution when you debug a system with heavy traffic. Use an ACL in order to debug specific traffic. Ensure the process and the traffic flow.

-
Use the

**access-list**command in order to capture the desired data.In this example, the data capture is set for the destination address of 10.2.6.6 or the source address of 10.2.6.6.

**access-list 101 permit ip any host 10.2.6.6 access-list 101 permit ip host 10.2.6.6 any** -
Disable fast switching on the interfaces involved. You only see the first packet if fast switching is not disabled.

**configure terminal**

interface <interface-name> no ip route-cache -
Use the

**terminal monitor**command in enable mode in order to display**debug**command output and system error messages for the current terminal and session. -
Use the

**debug ip packet 101**or**debug ip packet 101 detail**command in order to begin the debug process. -
Execute the

**no debug all**command in enable mode and the**interface configuration**command in order to stop the debug process. -
Restart caching.

**configure terminal**

interface <interface-name> ip route-cache

This section of the document describes ACL types.

Standard ACLs are the oldest type of ACL. They date back to as early as Cisco IOS Software Release 8.3. Standard ACLs control traffic by the comparison of the source address of the IP packets to the addresses configured in the ACL.

This is the command syntax format of a standard ACL.

access-list <access-list-number> {permit|deny} {host|source source-wildcard|any}

In all software releases, the access-list-number can be anything from 1 to 99. In Cisco IOS Software Release 12.0.1, standard ACLs begin to use additional numbers (1300 to 1999). These additional numbers are referred to as expanded IP ACLs. Cisco IOS Software Release 11.2 added the ability to use list name in standard ACLs.

A source/source-wildcard setting of 0.0.0.0/255.255.255.255 can be specified as any . The wildcard can be omitted if it is all zeros. Therefore, host 10.1.1.2 0.0.0.0 is the same as host 10.1.1.2.

After the ACL is defined, it must be applied to the interface (inbound or outbound). In early software releases, out was the default when a keyword out or in was not specified. The direction must be specified in later software releases.

interface <interface-name>ip access-group <acl> {in|out}

This is an example of the use of a standard ACL in order to block all traffic except that from source 10.1.1.x.

interface Ethernet0/0 ip address 10.1.1.1 255.255.255.0 ip access-group 1 in !

access-list 1 permit 10.1.1.0 0.0.0.255

Extended ACLs were introduced in Cisco IOS Software Release 8.3. Extended ACLs control traffic by the comparison of the source and destination addresses of the IP packets to the addresses configured in the ACL.

This is the command syntax format of extended ACLs. Lines are wrapped here for space considerations.

access-listaccess-list-number[dynamicdynamic-name[timeoutminutes]] {deny|permit}protocol source source-wildcard destination destination-wildcard[precedenceprecedence] [tostos] [log|log-input] [time-rangetime-range-name]

access-listaccess-list-number[dynamicdynamic-name[timeoutminutes]] {deny|permit} icmpsource source-wildcard destination destination-wildcard[icmp-type [icmp-code] |icmp-message] [precedenceprecedence] [tostos] [log|log-inp

[... 内容超长，已截断；完整原文见 source URL ...]
