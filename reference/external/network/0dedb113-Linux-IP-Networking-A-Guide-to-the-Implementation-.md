---
title: Linux IP Networking A Guide to the Implementation and Modification of the Linux Protocol Stack
source: https://www.cs.unh.edu/cnrg/people/gherrin/linux-net.html#tth_chAp1
kind: external
domain: network
original_date: 2014-02-02
fetched_at: 2026-05-16
bookmark_title: Linux IP Networking: A Guide to the Implementation and Modification of the Linux Protocol Stack
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cs.unh.edu](https://www.cs.unh.edu/cnrg/people/gherrin/linux-net.html#tth_chAp1)
> 原始日期：2014-02-02
> 抓取日期：2026-05-16

# Linux IP Networking A Guide to the Implementation and Modification of the Linux Protocol Stack

A Guide to the Implementation and Modification of the Linux Protocol Stack

**Abstract**

This document is a guide to understanding how the Linux kernel (version 2.2.14 specifically) implements networking protocols, focused primarily on the Internet Protocol (IP). It is intended as a complete reference for experimenters with overviews, walk-throughs, source code explanations, and examples. The first part contains an in-depth examination of the code, data structures, and functionality involved with networking. There are chapters on initialization, connections and sockets, and receiving, transmitting, and forwarding packets. The second part contains detailed instructions for modifiying the kernel source code and installing new modules. There are chapters on kernel installation, modules, the *proc* file system, and a complete example.


1.1 Background

1.2 Document Conventions

1.3 Sample Network Example

1.4 Copyright, License, and Disclaimer

1.5 Acknowledgements

2 Message Traffic Overview

2.1 The Network Traffic Path

2.2 The Protocol Stack

2.3 Packet Structure

2.4 Internet Routing

3 Network Initialization

3.1 Overview

3.2 Startup

3.2.1 The Network Initialization Script

3.2.2

3.2.3

3.2.4 Dynamic Routing Programs

3.3 Examples

3.3.1 Home Computer

3.3.2 Host Computer on a LAN

3.3.3 Network Routing Computer

3.4 Linux and Network Program Functions

3.4.1

3.4.2

4 Connections

4.1 Overview

4.2 Socket Structures

4.3 Sockets and Routing

4.4 Connection Processes

4.4.1 Establishing Connections

4.4.2 Socket Call Walk-Through

4.4.3 Connect Call Walk-Through

4.4.4 Closing Connections

4.4.5 Close Walk-Through

4.5 Linux Functions

5 Sending Messages

5.1 Overview

5.2 Sending Walk-Through

5.2.1 Writing to a Socket

5.2.2 Creating a Packet with UDP

5.2.3 Creating a Packet with TCP

5.2.4 Wrapping a Packet in IP

5.2.5 Transmitting a Packet

5.3 Linux Functions

6 Receiving Messages

6.1 Overview

6.2 Receiving Walk-Through

6.2.1 Reading from a Socket (Part I)

6.2.2 Receiving a Packet

6.2.3 Running the Network ``Bottom Half''

6.2.4 Unwrapping a Packet in IP

6.2.5 Accepting a Packet in UDP

6.2.6 Accepting a Packet in TCP

6.2.7 Reading from a Socket (Part II)

6.3 Linux Functions

7 IP Forwarding

7.1 Overview

7.2 IP Forward Walk-Through

7.2.1 Receiving a Packet

7.2.2 Running the Network ``Bottom Half''

7.2.3 Examining a Packet in IP

7.2.4 Forwarding a Packet in IP

7.2.5 Transmitting a Packet

7.3 Linux Functions

8 Basic Internet Protocol Routing

8.1 Overview

8.2 Routing Tables

8.2.1 The Neighbor Table

8.2.2 The Forwarding Information Base

8.2.3 The Routing Cache

8.2.4 Updating Routing Information

8.3 Linux Functions

9 Dynamic Routing with

9.1 Overview

9.2 How

9.2.1 Data Structures

9.2.2 Initialization

9.2.3 Normal Operations

9.3

10 Editing Linux Source Code

10.1 The Linux Source Tree

10.2 Using EMACS Tags

10.2.1 Referencing with TAGS

10.2.2 Constructing TAGS files

10.3 Using vi tags

10.4 Rebuilding the Kernel

10.5 Patching the Kernel Source

11 Linux Modules

11.1 Overview

11.2 Writing, Installing, and Removing Modules

11.2.1 Writing Modules

11.2.2 Installing and Removing Modules

11.3 Example

12 The

12.1 Overview

12.2 Network

12.3 Registering

12.3.1 Formatting a Function to Provide Information

12.3.2 Building a

12.3.3 Registering a

12.3.4 Unregistering a

12.4 Example

13 Example - Packet Dropper

13.1 Overview

13.2 Considerations

13.3 Experimental Systems and Benchmarks

13.4 Results and Preliminary Analysis

13.4.1 Standard Kernel

13.4.2 Modified Kernel Dropping Packets

13.4.3 Preliminary Analysis

13.5 Code

13.5.1 Kernel

13.5.2 Module

14 Additional Resources

14.1 Internet Sites

14.2 Books

15 Acronyms


Introduction


This document is an effort to bring together many of these sources into one coherent reference on and guide to modifying the networking code within the Linux kernel. It presents the internal workings on four levels: a general overview, more specific examinations of network activities, detailed function walk-throughs, and references to the actual code and data structures. It is designed to provide as much or as little detail as the reader desires. This guide was written specifically about the Linux 2.2.14 kernel (which has already been superseded by 2.2.15) and many of the examples come from the Red Hat 6.1 distribution; hopefully the information provided is general enough that it will still apply across distributions and new kernels. It also focuses almost exclusively on TCP/UDP, IP, and Ethernet - which are the most common but by no means the only networking protocols available for Linux platforms.

As a reference for kernel programmers, this document includes information and pointers on editing and recompiling the kernel, writing and installing modules, and working with the */proc* file system. It also presents an example of a program that drops packets for a selected host, along with analysis of the results. Between the descriptions and the examples, this should answer most questions about how Linux performs networking operations and how you can modify it to suit your own purposes.

This project began in a Computer Science Department networking lab at the University of New Hampshire as an effort to institute changes in the Linux kernel to experiment with different routing algorithms. It quickly became apparent that blindly hacking the kernel was not a good idea, so this document was born as a research record and a reference for future programmers. Finally it became large enough (and hopefully useful enough) that we decided to generalize it, formalize it, and release it for public consumption.

As a final note, Linux is an ever-changing system and truly mastering it, if such a thing is even possible, would take far more time than has been spent putting this reference together. If you notice any misstatements, omissions, glaring errors, or even typos (!) within this document, please contact the person who is currently maintaining it. The goal of this project has been to create a freely available and useful reference for Linux programmers.


Almost all of the code presented requires superuser access to implement. Some of the examples can create security holes where none previously existed; programmers should be careful to restore their systems to a normal state after experimenting with the kernel.

File references and program names are written in a *slanted* font.

Code, command line entries, and machine names are written in a `typewriter` font.

Generic entries or variables (such as an output filename) and comments are written in an *italic* font.



This network represents the computer system at a fictional unnamed University (U!). It has a router connected to the Internet at large (`chrysler`). That machine is connected (through the `jeep` interface) to the campus-wide network, `u.edu`, consisting of computers named for Chrysler owned car companies (`dodge`, `eagle`, etc.). There is also a LAN subnet for the computer science department, `cs.u.edu`, whose hosts are named after Dodge vehicle models (`stealth`, `neon`, etc.). They are connected to the campus network by the `dodge/viper` computer. Both the `u.edu` and `cs.u.edu` networks use Ethernet hardware and protocols.

This is obviously not a real network. The IP addresses are all taken from the block reserved for class B private networks (that are not guaranteed to be unique). Most real class B networks would have many more computers, and a network with only eight computers would probably not have a subnet. The connection to the Internet (through `chrysler`) would usually be via a T1 or T3 line, and that router would probably be a ``real'' router (i.e. a Cisco Systems hardware router) rather than a computer with two network cards. However, this example is realistic enough to serve its purpose: to illustrate the the Linux network implementation and the interactions between hosts, subnets, and networks.


Copyright (c) 2000 by Glenn Herrin. This document may be freely reproduced in whole or in part provided credit is given to the author with a line similar to the following:

(The visibility of the credit should be proportional to the amount of the document reproduced!) Commercial redistribution is permitted and encouraged. All modifications of this document, including translations, anthologies, and partial documents, must meet the following requirements:Copied from Linux IP Networking, available at.http://www.cs.unh.edu/cnrg/gherrin

- Modified versions must be labeled as such.
- The person making the modifications must be identified.
- Acknowledgement of the original author must be retained.
- The location of the original unmodified document be identified.
- The original author's name may not be used to assert or imply endorsement of the resulting document without the original author's permission.

Please note any modifications including deletions.

This is a variation (changes are intentional) of the Linux Documentaion Project (LDP) License available at:

This document is not currently part of the LDP, but it may be submitted in the future.http://www.linuxdoc.org/COPYRIGHT.html

This document is distributed in the hope that it will be useful but (of course)without any given or implied warranty of fitness for any purpose whatsoever. Use it at your own risk.


Glenn Herrin

Major, United States Army

Primary Documenter and Researcher, Version 1.0

gherrin@cs.unh.edu


Message Traffic Overview

This chapter presents an overview of the entire Linux messaging system. It provides a discussion of configurations, introduces the data structures involved, and describes the basics of IP routing.



When an application generates traffic, it sends packets through sockets to a transport layer (TCP or UDP) and then on to the network layer (IP). In the IP layer, the kernel looks up the route to the host in either the routing cache or its Forwarding Information Base (FIB). If the packet is for another computer, the kernel addresses it and then sends it to a link layer output interface (typically an Ethernet device) which ultimately sends the packet out over the physical medium.

When a packet arrives over the medium, the input interface receives it and checks to see if the packet is indeed for the host computer. If so, it sends the packet up to the IP layer, which looks up the route to the packet's destination. If the packet has to be forwarded to another computer, the IP layer sends it back down to an output interface. If the packet is for an application, it sends it up through the transport layer and sockets for the application to read when it is ready.

Along the way, each socket and protocol performs various checks and formatting functions, detailed in later chapters. The entire process is implemented with references and jump tables that isolate each protocol, most of which are set up during initialization when the computer boots. See Chapter 3 for details of the initialization process.


IP is the standard network layer protocol. It checks incoming packets to see if they are for the host computer or if they need to be forwarded. It defragments packets if necessary and delivers them to the transport protocols. It maintains a database of routes for outgoing packets; it addresses and fragments them if necessary before sending them down to the link layer.

TCP and UDP are the most common transport layer protocols. UDP simply provides a framework for addressing packets to ports within a computer, while TCP allows more complex connection based operations, including recovery mechanisms for packet loss and traffic management implementations. Either one copies the packet's payload between user and kernel space. However, both are just part of the intermediate layer between the applications and the network.

IP Specific INET Sockets are the data elements and implementations of generic sockets. They have associated queues and code that executes socket operations such as reading, writing, and making connections. They act as the intermediary between an application's generic socket and the transport layer protocol.

Generic BSD Sockets are more abstract structures that contain INET sockets. Applications read from and write to BSD sockets; the BSD sockets translate the operations into INET socket operations. See Chapter 4 for more on sockets.

Applications, run in user space, form the top level of the protocol stack; they can be as simple as two-way chat connection or as complex as the Routing Information Protocol (RIP - see Chapter 9).



This structure contains pointers to all of the information about a packet - its socket, device, route, data locations, etc. Transport protocols create these packet structures from output buffers, while device drivers create them for incoming data. Each layer then fills in the information that it needs as it processes the packet. All of the protocols - transport (TCP/UDP), internet (IP), and link level (Ethernet) - use the same socket buffer.


The FIB is the primary routing reference; it contains up to 32 zones (one for each bit in an IP address) and entries for every known destination. Each zone contains entries for networks or hosts that can be uniquely identified by a certain number of bits - a network with a netmask of 255.0.0.0 has 8 significant bits and would be in zone 8, while a network with a netmask of 255.255.255.0 has 24 significant bits and would be in zone 24. When IP needs a route, it begins with the most specific zones and searches the entire table until it finds a match (there should always be at least one default entry). The file */proc/net/route* has the contents of the FIB.

The routing cache is a hash table that IP uses to actually route packets. It contains up to 256 chains of current routing entries, with each entry's position determined by a hash function. When a host needs to send a packet, IP looks for an entry in the routing cache. If there is none, it finds the appropriate route in the FIB and inserts a new entry into the cache. (This entry is what the various protocols use to route, not the FIB entry.) The entries remain in the cache as long as they are being used; if there is no traffic for a destination, the entry times out and IP deletes it. The file */proc/net/rt_cache* has the contents of the routing cache.

These tables perform all the routing on a normal system. Even other protocols (such as RIP) use the same structures; they just modify the existing tables within the kernel using the `ioctl()` function. See Chapter 8 for routing details.


Network Initialization

This chapter presents network initialization on startup. It provides an overview of what happens when the Linux operating system boots, shows how the kernel and supporting programs *ifconfig* and *route* establish network links, shows the differences between several example configurations, and summarizes the implementation code within the kernel and network programs.


The entire configuration process can be static or dynamic. If addresses and names never (or infrequently) change, the system administrator must define options and variables in files when setting up the system. In a more mutable environment, a host will use a protocol like the Dynamic Hardware Configuration Protocol (DHCP) to ask for an address, router, and DNS server information with which to configure itself when it boots. (In fact, in either case, the administrator will almost always use a GUI interface - like Red Hat's Control Panel - which automatically writes the configuration files shown below.)

An important point to note is that while most computers running Linux start up the same way, the programs and their locations are not by any means standardized; they may vary widely depending on distribution, security concerns, or whim of the system administrator. This chapter presents as generic a description as possible but assumes a Red Hat Linux 6.1 distribution and a generally static network environment.



The script(s) involved in establishing networking can be very straightforward; it is entirely possible to have one big script that simply executes a series of commands that will set up a single machine properly. However, most Linux distributions come with a large number of generic scripts that work for a wide variety of machine setups. This leaves a lot of indirection and conditional execution in the scripts, but actually makes setting up any one machine much easier. For example, on Red Hat distributions, the */etc/rc.d/init.d/network* script runs several other scripts and sets up variables like `interfaces_boot` to keep track of which */etc/sysconfig/network-scripts/ifup* scripts to run. Tracing the process manually is very complicated, but simple modifications of only two configuration files (putting the proper names and IP addresses in the */etc/sysconfig/network* and */etc/sysconfig/network-scripts/ifcfg-eth0* files) sets up the entire system properly (and a GUI makes the process even simpler).

When the network script finishes, the FIB contains the specified routes to given hosts or networks and the routing cache and neighbor tables are empty. When traffic begins to flow, the kernel will update the neighbor table and routing cache as part of the normal network operations. (Network traffic may begin during initialization if a host is dynamically configured or consults a network clock, for example.)


(where the variables are either written directly in the script or are defined in other scripts).ifconfig ${DEVICE} ${IPADDR} netmask ${NMASK} broadcast ${BCAST}

The *ifconfig* program can also provide information about currently configured network devices (calling with no arguments displays all the active interfaces; calling with the `-a` option displays all interfaces, active or not):

This provides all the information available about each working interface; addresses, status, packet statistics, and operating system specifics. Usually there will be at least two interfaces - a network card and the loopback device. The information for each interface looks like this (this is theifconfig

A superuser can useeth0 Link encap:Ethernet HWaddr 00:C1:4E:7D:9E:25 inet addr:172.16.1.1 Bcast:172.16.1.255 Mask:255.255.255.0 UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1 RX packets:389016 errors:16534 dropped:0 overruns:0 frame:24522 TX packets:400845 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:100 Interrupt:11 Base address:0xcc00

... and some of the more useful calls:ifconfiginterface [aftype] options | address ...

Note that modifying an interface configuration can indirectly change the routing tables. For example, changing the netmask may make some routes moot (including the default or even the route to the host itself) and the kernel will delete them.ifconfig eth0 down- shut downeth0

ifconfig eth1 up- activateeth1

ifconfig eth0 arp- enable ARP oneth0

ifconfig eth0 -arp- disable ARP oneth0

ifconfig eth0 netmask 255.255.255.0- set theeth0netmask

ifconfig lo mtu 2000- set the loopback maximum transfer unit

ifconfig eth1 172.16.0.7- set theeth1IP address


(where the variables are again spelled out or defined in other scripts).route add -net ${NETWORK} netmask ${NMASK} dev ${DEVICE}-or-

route add -host ${IPADDR} ${DEVICE}


The *route* program can also delete routes (if run with the `del` option) or provide information about the routes that are currently defined (if run with no options):

This displays the Kernel IP routing table (the FIB, not the routing cache). For example (theroute

A superuser can useKernel IP routing table Destination Gateway Genmask Flags Metric Ref Use Iface 172.16.1.4 * 255.255.255.255 UH 0 0 0 eth0 172.16.1.0 * 255.255.255.0 U 0 0 0 eth0 127.0.0.0 * 255.0.0.0 U 0 0 0 lo default viper.u.edu 0.0.0.0 UG 0 0 0 eth0

... and some useful examples:route add[-net|-host] target [option arg]

route del[-net|-host] target [option arg]

route add -host 127.16.1.0 eth1- adds a route to a host

route add -net 172.16.1.0 netmask 255.255.255.0 eth0- adds a network

route add default gw jeep- sets the default route throughjeep

(Note that a route tojeepmust already be set up)

route del -host 172.16.1.16- deletes entry for host172.16.1.16




This is the first file the network script will read; it sets several environment variables. The first two variables set the computer to run networking programs (even though it is not on a network) but not to forward packets (since it has nowhere to send them). The last two variables are generic entries.

*/etc/sysconfig/network*

After setting these variables, the network script will decide that it needs to configure at least one network device in order to be part of a network. The next file (which is almost exactly the same on all Linux computers) sets up environment variables for the loopback device. It names it and gives it its (standard) IP address, network mask, and broadcast address as well as any other device specific variables. (The ONBOOT variable is a flag for the script program that tells it to configure this device when it boots.) Most computers, even those that will never connect to the Internet, install the loopback device for inter-process communication.NETWORKING=yes

FORWARD_IPV4=false

HOSTNAME=localhost.localdomain

GATEWAY=

*/etc/sysconfig/network-scripts/ifcfg-lo*

After setting these variables, the script will run theDEVICE=lo

IPADDR=127.0.0.1

NMASK=255.0.0.0

NETWORK=127.0.0.0

BCAST=127.255.255.255

ONBOOT=yes

NAME=loopback

BOOTPROTO=none


This is the first file the network script will read; again the first variables simply determine that the computer will do networking but that it will not forward packets. The last four variables identify the computer and its link to the rest of the Internet (everything that is not on the LAN).

*/etc/sysconfig/network*

After setting these variables, the network script will configure the network devices. This file sets up environment variables for the Ethernet card. It names the device and gives it its IP address, network mask, and broadcast address as well as any other device specific variables. This kind of computer would also have a loopback configuration file exactly like the one for a non-networked computer.NETWORKING=yes

FORWARD_IPV4=false

HOSTNAME=stealth.cs.u.edu

DOMAINNAME=cs.u.edu

GATEWAY=172.16.1.1

GATEWAYDEV=eth0

*/etc/sysconfig/network-scripts/ifcfg-eth0*

DEVICE=eth0

IPADDR=172.16.1.4

NMASK=255.255.255.0

NETWORK=172.16.1.0

BCAST=172.16.1.255

ONBOOT=yes

BOOTPROTO=none

After setting these variables, the network script will run the *ifconfig program* to start the device. Finally, the script will run the *route* program to add the default route (`GATEWAY`) and any other specified routes (found in the */etc/sysconfig/static-routes file*, if any). In this case only the default route is specified, since all traffic either stays on the LAN (where the computer will use ARP to find other hosts) or goes through the router to get to the outside world.


This is the first file the network script will read; it sets several environment variables. The first two simply determine that the computer will do networking (since it is on a network) and that this one will forward packets (from one network to the other). IP Forwarding is built into most kernels, but it is not active unless there is a 1 ``written'' to the */proc/net/ipv4/ip_forward* file. (One of the network scripts performs an `echo 1 > /proc/net/ipv4/ip_forward` if `FORWARD_IPV4` is true.) The last four variables identify the computer and its link to the rest of the Internet (everything that is not on one of its own networks).

*/etc/sysconfig/network*

After setting these variables, the network script will configure the network devices. These files set up environment variables for two Ethernet cards. They name the devices and give them their IP addresses, network masks, and broadcast addresses. (Note that the BOOTPROTO variable remains defined for the second card.) Again, this computer would have the standard loopback configuration file.NETWORKING=yes

FORWARD_IPV4=true

HOSTNAME=dodge.u.edu

DOMAINNAME=u.edu

GATEWAY=172.16.0.1

GATEWAYDEV=eth1

*/etc/sysconfig/network-scripts/ifcfg-eth0*

DEVICE=eth0

IPADDR=172.16.1.1

NMASK=255.255.255.0

NETWORK=172.16.1.0

BCAST=172.16.1.255

ONBOOT=yes

BOOTPROTO=static

*/etc/sysconfig/network-scripts/ifcfg-eth1*

After setting these variables, the network script will run theDEVICE=eth1

IPADDR=172.16.0.7

NMASK=255.255.0.0

NETWORK=172.16.0.0

BCAST=172.16.255.255

ONBOOT=yes


These sources are available as a package separate from the kernel source (Red Hat Linux uses the *rpm* package manager). The code below is from the *net-tools-1.53-1* source code package, 29 August 1999. The packages are available from the *www.redhat.com/apps/download* web page. Once downloaded, *root* can install the package with the following commands (starting from the directory with t

[... 内容超长，已截断；完整原文见 source URL ...]
