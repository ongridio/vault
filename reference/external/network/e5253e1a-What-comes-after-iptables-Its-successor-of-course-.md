---
title: What comes after 'iptables'? Its successor, of course: `nftables` | Red Hat Developer
source: https://developers.redhat.com/blog/2016/10/28/what-comes-after-iptables-its-successor-of-course-nftables/
kind: external
domain: network
author: Florian Westphal
original_date: 2016-10-28
fetched_at: 2026-05-16
bookmark_title: What comes after 'iptables'? Its successor, of course: `nftables` - Red Hat Developer
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[developers.redhat.com](https://developers.redhat.com/blog/2016/10/28/what-comes-after-iptables-its-successor-of-course-nftables/)
> 作者：Florian Westphal
> 原始日期：2016-10-28
> 抓取日期：2026-05-16

# What comes after 'iptables'? Its successor, of course: `nftables` | Red Hat Developer

Nftables is a new packet classification framework that aims to replace the existing iptables, ip6tables, arptables and ebtables facilities. It aims to resolve a lot of limitations that exist in the venerable ip/ip6tables tools. The most notable capabilities that nftables offers over the old iptables are:

#### Performance:

- Support for lookup tables - no linear rule evaluation required
- No longer enforces overhead of implicit rule counters and address/interface matching

#### Usability:

- Transactional rule updates - all rules are applied atomically
- Applications can subscribe to nfnetlink notifications to receive rule updates when new rules get added or removed
- nft command-line tool can display a live-log of rules that are being matched for easier ruleset debugging

## What is nftables?

Nftables was initially presented at the Netfilter Workshop 2008 in Paris, France and then released in March 2009 by long-time netfilter core team member and project lead, Patrick McHardy. It was merged into the Linux kernel in late 2013 and has been available since 2014 with kernel 3.13.

It re-uses many parts of the netfilter framework, such as the connection tracking and NAT facilities. It also retains several parts of the nomenclature and basic iptables design, such as tables, chains and rules. Just like with iptables, tables serve as containers for chains, and the chains contain individual rules that can perform actions such as dropping a packet, moving to the next rule, or perform a jump to a new chain.

## What is being replaced?

From a user point of view, nftables adds a new tool, called nft, which replaces all other tools from iptables, arptables and ebtables. From an architectural point of view it also replaces those parts of the kernel that deal with run-time evaluation of the packet filtering rule set.

## Why replace iptables?

Despite its success, iptables has several architectural limitations that cannot be resolved without many fundamental design changes. The main problem stems from the fact that the ruleset representation is shared by the frontend tools like iptables and the kernel. This has multiple drawbacks. When a rule is added to an iptables ruleset, for example

# allow inbound ssh connections iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT

Then the iptables tool first needs to dump the current ruleset -- essentially an array of various data structures -- then modifies this blob to contain the new rule, then send the entire set of rules back to the kernel. This also means that every add and remove operation is slow, as the entire table has to be processed and then replaced with a new one.

**The kernel has no idea what rule got modified, added or removed, from the ruleset in the table being replaced.**

All the kernel sees is that a new table ruleset is being loaded, it cannot detect which part of the rule set was removed or added, and which part of the ruleset has changed as a side effect (such as changed offsets when a rule jumps to a different chain).

**Userspace has no idea what the ruleset is doing.**

All the iptables tool provides is parsing of the command line arguments and the translation to the binary representation that the kernel expects. Userspace has no notion of ''IP'' or ''TCP''. This also prevents rule set optimization steps in userspace without replicating the evaluation logic implemented in the kernel.

**Addition and removal of rules is not an atomic operation.**

If two rules are added at the exact same time, only one of those will take effect, as each process independently manipulates the current table, then asks kernel to swap the old with a new one -- so the process that comes last will remove any changes made in-between its dump and commit operation. This was not a problem when iptables and its predecessor, ipchains, was originally developed at the end of the 1990s, as only the system administrator was expected to change packet filter rules. Nowadays this has changed. For example applications like docker or libvirt want to add NAT rules to provide connectivity for containers and virtual machines.

**The iptables rule set format is set in stone forever.**

Aside from a ''revision number'' scheme that can be used to extend functionality provided by matches and targets the format of the rule blob cannot be altered without breaking compatibility. The current format imposes several restrictions:

The kernel always performs matching of ip source and destination address, even if no address was given (the kernel will compare packet source and destination with 0.0.0.0/0 which will always succeeed). The input and output interface is also always tested for each packet, because like IP addresses these are stored in the header location of each rule. A counter is incremented whenever a rule matches. This is nice from a user’s perspective, as it allows for an easy and simple way to check if rules are being evaluated as expected, but this does become a performance problem when millions of packets are processed per second.

**An ever increasing number of special-features matches and targets.**

iptables comes with several 'matches' that implement various functionality to decide if a given packet is supposed to 'match' a given criterion.

There a some basic matches that e.g. allow matching on UDP or TCP port numbers from the packet payload, and several matches that perform more elaborate checks, for instance the conntrack match that queries the connection tracking state and can be used to e.g. test if a packet is a reply to earlier packets seen in the other direction, policy matching to evaluate if a given packet is subject to IPsec processing, or even if a packet is being processed on a particular CPU (useful for load-balancing).

When support for a new protocol has to be added both the kernel and iptables userspace need to be extended accordingly. iptables also has a number of more exotic targets -- for example the HMARK target to set a packet mark based on a hash function (added to support load balancing via policy routing). Another problem is that iptables doesn't allow for multiple actions (targets) to be executed in a rule. For instance it’s possible to create a rule that logs a packet, but we cannot -- in the same rule -- instruct the kernel to then also drop that packet. This makes it necessary to duplicate some rules when more than one action is desired, such as mark and accept.

**basic matching features are limited**

An iptables rule can only be used to match at most one source address (or network), and one destination address/network. It can also match against one source or destination interface only. To match against a list or range of addresses, external facilties such as the ''ipset'' tool need to be used.

nftables is designed to overcome all these limitations:

- Support sets and maps to allow compact representation of rules rather than forcing a linear top-down evaluation approach as with iptables.
- Addition and removal of rules is always atomic, no races happen when two programs add/remove rules simultaneously, i.e. userspace deletes and adds individual rules instead of the entire table.
- The kernel provides a unique identification number for each individual rule which makes deletion of rules a lot simpler than with iptables
- Applications can ask the kernel to get a notification when rules are added or removed.
- No distinction between matches and targets anymore. nftables uses what it calls 'expressions' to implement functionality. It is possible to e.g. set a mark on a packet and accept it in the same rule.
- Move most rule handling to userspace. While processing remains in the kernel, protocols are now handled in userspace.For instance, the kernel no longer knows how to load the TCP destination port, an IP source address, or the SPI from ESP headers. Instead, the nft tool would ask the kernel to load a number of bytes starting at a particular offset.Adding support for new transport headers therefore no longer needs kernelchanges, the nft tool only needs to be extended to know about the header fields,their names, sizes and location.
- Allow monitoring of rule updates. The nft tool can be used to monitor changes in the rule set, e.g.''nft monitor'' displays every rule added and removed from the kernel.It also offers a 'trace mode' where a system administrator can for instance do 'nft add rule mangle prerouting ip saddr 10.2.3.4 meta nftrace set 1'. ''nft monitor trace'' will then display every rule that gets matched by packets originating from the IP address 10.2.3.4.

## Nftables at a high level

nftables implements a set of instructions, called expressions, which can exchange data by storing or loading it in a number of registers. In other words, the nftables core can be seen as a virtual machine. Applications like the nftables front end-tool nft can use the expressions offered by the kernel to mimic the old iptables matches while gaining more flexibility. For example, when nft is asked to check if a packet matches a given TCP destination port, it instructs the kernel to load 2 bytes at offset 2 from the start of the TCP header into a given register and then to compare that register with an immediate value (i.e. the port number):

# nft add rule ip filter input tcp dport 22

To better understand what is happening behind the scenes, one can pass the

--debug=netlink

option to nft. Doing this with the above statement would yield

the following debug output which shows the generated individual expressions that make up the rule:

[ payload load 1b @ network header + 9 => reg 1 ] [ cmp eq reg 1 0x00000006 ] [ payload load 2b @ transport header + 2 => reg 1 ] [ cmp eq reg 1 0x00001600 ]

So when the rule is evaluated in the kernel, it will first load the IP protocol field into register 1, then compare it with the number 6 (to check if the transport header is TCP), then load the TCP destination port, then check if it matches the number 22 (0x1600 shown here is the hexadecimal display of 22 in network byte order). The kernel is agnostic to the protocol details -- it only loads data into registers and performs some comparisons.

This register approach means that most high-level functionality is achieved by chaining these basic building blocks in some way. Most expressions either take data from registers or know how to store something in a register, others only have side-effects.

Examples of expressions that don't use registers:

- log: causes the kernel to emit a log entry, either using the kernel ring buffer (like old iptables -j LOG or nflog, like iptables -j NFLOG).
- limit: often combined with the log expression to avoid log flooding, can also be used to implement rate policing.

Examples of expressions that store data in registers:

- payload: loads data from the packet payload and places them in registers.
- immediate: store a given number in a given register. This is for example used to implement the accept and drop statements in the kernel -- when user would ask to drop a packet, nft will use the immediate expression to store the verdict code (accept, drop, ...) in the verdict register.
- meta: Ancillary data, such as the name of the input or output interface names the packet arrived on, the skb mark, the length/size of the packet, ...
- ct: Provides access to conntrack related data, such as the packet’s direction (original, reply), the conntrack counters (bytes/packets seen so far), the state (new, established, ...)

Some of these expressions also allow setting a few attributes, for instance the payload expression can also do the reverse -- take a register content and place it into the packet at a certain location, the meta expression can set the packet mark, and so on.

Other expressions can be used to manipulate register content.

For example, if user would ask not to match the exact IP address 10.0.0.0 but the network 10.0.0.0/8, then nft tool would ask to load the IP address into a register, use the 'bitwise' expression to only retain the upper part of the address (10), and compare the register with the immediate value 10.

The same is true when e.g. asking to match interface names starting with a particular prefix, e.g. 'ppp'.

Other expressions perform tests on a register:

- cmp: checks if the content of a register is less, equal or greater than a given immediate value. This also means that kernel doesn't need to support e.g. matching IP address ranges -- userspace simply specifies two compare operations:

reg1 >= 0xc0a80010 and reg1 <= 0xc0a80280

to determine if the IP address placed into the register is within an address range.

- lookup: checks if register content is present in a given set. nftables supports sets of data, i.e. a given number of addresses, port numbers, interface names and so on.

For example, in nftables one can create a rule that matches when the TCP destination port is contained in a set of values:

# nft add rule ip filter input tcp dport { 80, 443, 8080 }

Internally nft creates a new set, containing the three given numbers as keys, then asks the nftables machine to check if it can find a result:

[ payload load 1b @ network header + 9 => reg 1 ] [ cmp eq reg 1 0x00000006 ] [ payload load 2b @ transport header + 2 => reg 1 ] [ lookup reg 1 set __set%d ]

These are almost the same generated expressions as before, we check if the packet is a TCP packet, but instead of comparing the register with an immediate value, we ask the check if the register is contained in a set. A set can not only store keys -- it can also store key and value pairs.

The lookup expression can also retrieve the values from such key/value sets, i.e. userspace can create a table mapping keys to values and then ask to fill a register with content retrieved from that set. This can be used for instance to implement branching (verdict maps) in the ruleset.

## Verdict maps (aka jump tables)

While an iptables rule can tell the kernel to jump to a custom chain rather than moving to the next rule, it is not possible to use branching. For instance, let us consider the following:

iptables -N MGMT_PACKETS iptables -N VM1_PACKETS iptables -N VM2_PACKETS iptables -A FORWARD -i eth0 -s 192.168.0.1 -j MGMT_PACKETS iptables -A FORWARD -i tap0 -s 10.0.0.1 -j VM1_PACKETS iptables -A FORWARD -i tap1 -s 10.0.0.2 -j VM1_PACKETS

This adds three new chains, then tells the kernel to evaluate packets being forwarded and coming from specific combinations of source addresses and input interfaces to a chain for more specific processing. Since rule set evaluation is linear, the kernel has to check all these rules until it finds one that matches -- this will not scale when hundreds of interface and address combinations need to be considered.

What should be used here is a lookup table. With nftables, it is possible to express the above in a single rule:

chain forward { type ip filter hook forward priority 0; policy accept; iif . ip saddr vmap { eth0 . 192.168.0.1 : jump mgmt_packets, tap0 . 10.0.0.1 : jump vm1_packets, tap1 . 10.0.0.2 : jump vm2_packets } } chain vm1_packets {} chain vm2_packets {} chain local_packets {}

The nftables '.' syntax is used to concatenate two distinct sources together to obtain a single key: Here we tell nftables that we want to retrieve a jump target using a key that consists of the input interface and the IP source address. Now imagine the example with dozens or even hundreds of such 'jump to this chain if packet came from this given interface/address pair' -- with iptables this will simply not scale anymore. With nft, however, adding more jump targets to the lookup table has no extra processing cost except the extra memory useage needed to store the additonal information.

## Flow statement

Iptables offers a match named 'hashlimit'. It is like the 'limit' match, except that the limit can be keyed to certain properties, for example one can use the hash limit to provide a limit per source IP address, or even per source and destination pairs. To do this, the hashlimit match dynamically creates these rate-limit counters and stores them in a hash table -- hence the name.

Instead of just implementing a 1:1 replacement, nftables applies the same concept in a much more generic manner: rather than just allow to dynamically instantiate limits per IP addresses or networks, it allows the user to instantiate an expression and key it to an arbitrary set of criteria. Example to allow a similar effect than the hashlimit match in iptables:

nft add rule ip filter input flow table mylimit { ip saddr . tcp dport limit rate 10/second }

The flow statement also offers a timeout option to automatically remove flows once it has not been seen anymore for a given period. The same statement can also be used for per-host or service accounting:

nft add rule ip filter forward flow table ipacct { ip daddr counter } nft add rule ip filter forward flow table portcount { tcp dport counter }

Flow tables can be listed with the command

nft list flow tables

And its contents can be inspected with

nft list flow table filter ipacct table ip filter { flow table ipacct { type ipv4_addr elements = { 192.168.7.1 : counter packets 4 bytes 336, 192.168.0.7 : counter packets 10 bytes 688, 127.0.0.1 : counter packets 18 bytes 1512} } }

Here the flow table lists the seen IP addresses and their counters, which provides a simple way to perform per host traffic accounting.

## Inet family

To make it easier to create rule sets for host that use both IPv4 and IPv6 nftables comes with a new 'inet' family that can be used to create a single rule set used for both IPv4 and IPv6. This is done by using the inet keyword instead of the default ip one:

table inet filter { chain input { type filter hook input priority 0; policy accept; tcp dport ssh ip saddr 1.2.3.4 } }

This shows two rules. The first one will match both IPv4 and IPv6 traffic, as the rule only examines a protocol that is common for both (TCP in this case). The second rule tests an IP address. Because the rule doesn't reside in the ipv4 specific ip table, but in the ipv4 and ipv6 combined inet table, nftables has injected a so-called 'payload dependency'. One can

inet filter input 5 [ meta load l4proto => reg 1 ] [ cmp eq reg 1 0x00000006 ] [ payload load 2b @ transport header + 2 => reg 1 ] [ cmp eq reg 1 0x00001600 ] inet filter input 6 [ meta load nfproto => reg 1 ] [ cmp eq reg 1 0x00000002 ] [ payload load 4b @ network header + 12 => reg 1 ] [ cmp eq reg 1 0x04030201 ]

This shows two rules. The first one will match both IPv4 and IPv6 traffic, as the rule only examines a protocol that is common for both (TCP in this case). The second rule tests an IP address. Because the rule doesn't reside in the ipv4 specific ip table, but in the ipv4 and ipv6 combined inet table, nftables has injected a so-called 'payload dependency': It will first test that the packet is IPv4 (nfprotocol 2), and only then will it load the payload from the packet. This is required to prevent false-positive matches when packets from a different family are processed. This allows adding ipv4 and ipv6 specific rules to the inet table, or addition of a rule that test e.g. an ipv6 header to the bridge family. Payload dependencies are common. The first rule shown above also has one -- to prevent a false positive when processing e.g. a UDP packet to port 22 nftables injected a test on the layer 4 protocol -- 6, better known as tcp.

## Getting started

Users familiar with iptables, ip6tables, ebtables, arptables, iptables-save, iptables-restore and so on will notice that the syntax is completely different. But don't be discouraged -- there are a lot of conceptual similarities. It is possible to use both nftables and iptables at the same time. The only limitation is network address translation -- if iptable_nat and/or ip6table_nat modules are loaded, nftables “nat” chains will have no effect.

Use a recent distribution and install the 'nftables' package to get the 'nft' tool. nft is the only command that is needed to add/remove rules. nft can be used from the command line, in an interactive (shell-like) mode, or can restore a saved ruleset from a file. Once nft is available, one can start looking around -- this is best done in a vm or on a test machine rather than a production system:

# nft list tables

On a new system this would not provide any output -- there are no tables by default. One could now create a new table:

# nft add table ip foo # nft list tables table ip foo # nft list table foo table ip foo {}

As we can see, a new empty table named foo has been created, and it will only see ipv4 packets. Other families available in nftables are arp, ip6, bridge, inet and netdev.

- The arp family will only see ARP packets, the ip6 family will only process IPv6 packets.
- The bridge family only processes ethernet packets while tables in the
- inet family receive bothIPv4 and IPv6 packets. The
- netdev family can be used to attach a table directly to anetwork interface. Tables in the netdev family offer the new “ingress” hook point which can be used to filter any traffic received on that interface.

Other than the family, which defines what packet types will be processed, tables have no special meaning at all, they only serve as a container for chains. A table also defines a distinct namespace, so it is possible to create chains with the same name as long as these reside in different tables.


### Adding chains

Users familiar with iptables will remember that each table has built-in chains. In iptables these are called PREROUTING (for packets coming in), INPUT (for packets destined for the local machine), FORWARD (for packets that are being forwarded/routed), OUTPUT (for locally generated packets) and POSTROUTING for all packets being sent out.

Nftables has none of these built-in chains. However, it defines these very same hook-points. Hook points are certain locations in the kernel networking stack that provide the ability to inspect packets and provide decisions based on inspection results, such as drop or accept. These hooks are internally also used by iptables, ip6tables, ebtables and arptables to register their builtin chains. So, let’s add a chain that hooks packets at the input hook point:

# nft add chain ip foo bar '{ type filter hook input priority 0; }' # nft list table foo table ip foo { chain bar { type filter hook input priority 0; policy accept; } }

What did we do here? We told the kernel that we want to have packets destined for the local machine (input hook) to be sent through the rules contained in the chain named 'bar'. To mimic the iptables filter table, one would rename bar to INPUT and also create

chain FORWARD { type filter hook forward; policy accept; }

... and the same again for the output hook. Chains that include a hook name are called base chains. There can be several base chains, even with the same hook point. In this case, the chain with the lowest priority number (which can be negative a wel!l) is processed first. If base chains have the same priority, the order is undefined.

Let’s add a rule:

# nft add rule ip foo bar counter # nft list table foo table ip foo { chain bar { type filter hook input priority 0; policy accept; counter packets 2 bytes 168 } }

As can be seen, we now have one table with one chain hooked into the input path with one rule that just increments a counter with every packet seen. Now you might be wondering why we have to do all this work just to add one single rule, or what the 'type filter' means.

While it’s true that nftables does not have any builtin tables, it does ship with pre-created rule files that provide the nft-counterparts to the ip(6)tables built-in tables. Lets first undo our changes, then look at the mangle table as it would be created in nftables:

# nft delete table foo # cat /etc/nftables/ipv4-mangle #!/sbin/nft -f table mangle { chain output { type route hook output priority -150; } }

(Your distribution might have decided to place this elsewhere, e.g. in /usr/etc/nftables).

This table can be loaded via

# nft -f /etc/nftables/ipv4-mangle

The special 'route' type tells the netfilter core that it should perform a new route lookup after the packet has traversed the chain. This is mostly used with the packet mark and the 'ip rule' command to implement policy routing. In the old iptables world, this property is hard-coded into the mangle table; in nftables, it is a base chain propert. Any chain that will alter packet properties for the purpose of route changes need this keyword for the re-lookup to be triggered. Similar skeleton rulesets exist for the other tables that were built-in to the old generation of tools, e.g:

# ls /etc/nftables bridge-filter ipv4-filter ipv4-nat ipv6-filter ipv6-nat inet-filter ipv4-mangle ipv4-raw ipv6-mangle

So, let’s try again with something more useful and add a simple IPv4 and IPv6 filter set:

# nft -f /etc/nftables/inet-filter # nft add rule inet filter in

[... 内容超长，已截断；完整原文见 source URL ...]
