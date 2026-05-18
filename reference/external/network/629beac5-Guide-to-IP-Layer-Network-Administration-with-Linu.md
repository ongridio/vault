---
title: Guide to IP Layer Network Administration with Linux
source: http://linux-ip.net/linux-ip/linux-ip-single.html#nat-overview
kind: external
domain: network
author: Chandra Kopparapu
original_date: 2007-03-14
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[linux-ip.net](http://linux-ip.net/linux-ip/linux-ip-single.html#nat-overview)
> 作者：Chandra Kopparapu
> 原始日期：2007-03-14
> 抓取日期：2026-05-16

# Guide to IP Layer Network Administration with Linux

Copyright © 2002, 2003 Martin A. Brown

2007-Mar-14

**Abstract**

This guide provides an overview of many of the tools available for IP network administration of the linux operating system, kernels in the 2.2 and 2.4 series. It covers Ethernet, ARP, IP routing, NAT, and other topics central to the management of IP networks.

**Table of Contents**

- Introduction
- I. Concepts
- 1. Basic IP Connectivity
- 2. Ethernet
- 3. Bridging
- 4. IP Routing
- 5. Network Address Translation (NAT)
- 5.1. Rationale for and Introduction to NAT
- 5.2. Application Layer Protocols with Embedded Network Information
- 5.3. Stateless NAT with
**iproute2** - 5.4. Stateless NAT and Packet Filtering
- 5.5. Destination NAT with netfilter (DNAT)
- 5.6. Port Address Translation (PAT) from Userspace
- 5.7. Transparent PAT from Userspace

- 6. Masquerading and Source Network Address Translation
- 7. Packet Filtering
- 8. Statefulness and Statelessness

- II. Cookbook
- III. Appendices and Reference
- A. An Example Network and Description
- B. Ethernet Layer Tools
- B.1.
**arp** - B.2.
**arping** - B.3.
**ip link** - B.3.1. Displaying link layer characteristics with
**ip link show** - B.3.2. Changing link layer characteristics with
**ip link set** - B.3.3. Deactivating a device with
**ip link set** - B.3.4. Activating a device with
**ip link set** - B.3.5. Using
**ip link set**to change the MTU - B.3.6. Changing the device name with
**ip link set** - B.3.7. Changing hardware or Ethernet broadcast address with
**ip link set**

- B.3.1. Displaying link layer characteristics with
- B.4.
**ip neighbor** - B.5.
**mii-tool**

- B.1.
- C. IP Address Management
- D. IP Route Management
- D.1.
**route** - D.2.
**ip route** - D.2.1. Displaying a routing table with
**ip route show** - D.2.2. Displaying the routing cache with
**ip route show cache** - D.2.3. Using
**ip route add**to populate a routing table - D.2.4. Adding a default route with
**ip route add default** - D.2.5. Setting up NAT with
**ip route add nat** - D.2.6. Removing routes with
**ip route del** - D.2.7. Altering existing routes with
**ip route change** - D.2.8. Programmatically fetching route information with
**ip route get** - D.2.9. Clearing routing tables with
**ip route flush** - D.2.10.
**ip route flush cache** - D.2.11. Summary of the use of
**ip route**

- D.2.1. Displaying a routing table with
- D.3.
**ip rule**

- D.1.
- E. Tunnels and VPNs
- F. Sockets; Servers and Clients
- G. Diagnostic Tools
- H. Miscellany
- I. Links to other Resources
- I.1. Links to Documentation
- I.1.1. Linux Networking Introduction and Overview Material
- I.1.2. Linux Security and Network Security
- I.1.3. General IP Networking Resources
- I.1.4. Masquerading topics
- I.1.5. Network Address Translation
- I.1.6. iproute2 documentation
- I.1.7. Netfilter Resources
- I.1.8.
**ipchains**Resources - I.1.9.
**ipfwadm**Resources - I.1.10. General Systems References
- I.1.11. Bridging
- I.1.12. Traffic Control
- I.1.13. IPv4 Multicast
- I.1.14. Miscellaneous Linux IP Resources

- I.2. Links to Software

- J. GNU Free Documentation License
- J.1. PREAMBLE
- J.2. APPLICABILITY AND DEFINITIONS
- J.3. VERBATIM COPYING
- J.4. COPYING IN QUANTITY
- J.5. MODIFICATIONS
- J.6. COMBINING DOCUMENTS
- J.7. COLLECTIONS OF DOCUMENTS
- J.8. AGGREGATION WITH INDEPENDENT WORKS
- J.9. TRANSLATION
- J.10. TERMINATION
- J.11. FUTURE REVISIONS OF THIS LICENSE
- J.12. ADDENDUM: How to use this License for your documents


- Reference Bibliography and Recommended Reading
- Index

**List of Tables**

- 2.1. Active ARP cache entry states
- 4.1. Keys used for hash table lookups during route selection
- 5.1. Filtering an iproute2 NAT packet with ipchains
- A.1. Example Network; Network Addressing
- A.2. Example Network; Host Addressing
- B.1. ip link link layer device states
- B.2. Ethernet Port Speed Abbreviations
- C.1. Interface Flags
- C.2. IP Scope under ip address
- G.1. Possible Session States in netstat output
- H.1. iproute2 Synonyms

**List of Examples**

- 1.1. Sample ifconfig output
- 1.2. Testing reachability of a locally connected host with ping
- 1.3. Testing reachability of non-local hosts
- 1.4. Sample routing table with a static route
- 1.5. ifconfig and route output before the change
- 1.6. Bringing down a network interface with ifconfig
- 1.7. Bringing up an Ethernet interface with ifconfig
- 1.8. Adding a default route with route
- 1.9. Adding a static route with route
- 1.10. Removing a static network route and adding a static host route
- 2.1. ARP conversation captured with tcpdump
- 2.2. Gratuitous ARP reply frames
- 2.3. Unsolicited ARP request frames
- 2.4. Duplicate Address Detection with ARP
- 2.5. ARP cache listings with arp and ip neighbor
- 2.6. ARP cache timeout
- 2.7. ARP flux
- 2.8. Correction of ARP flux with
`conf/$DEV/arp_filter`

- 2.9. Correction of ARP flux with
`net/$DEV/hidden`

- 2.10. Proxy ARP Network Diagram
- 2.11. Bringing up a VLAN interface
- 2.12. Link aggregation bonding
- 2.13. High availability bonding
- 4.1. Classes of IP addresses
- 4.2. Using ipcalc to display IP information
- 4.3. Identifying the locally connected networks with route
- 4.4. Routing Selection Algorithm in Pseudo-code
- 4.5. Listing the Routing Policy Database (RPDB)
- 4.6. Typical content of
`/etc/iproute2/rt_tables`

- 4.7. unicast route types
- 4.8. broadcast route types
- 4.9. local route types
- 4.10. nat route types
- 4.11. unreachable route types
- 4.12. prohibit route types
- 4.13. blackhole route types
- 4.14. throw route types
- 4.15. Kernel maintenance of the
`local`

routing table - 4.16. unicast rule type
- 4.17. nat rule type
- 4.18. unreachable rule type
- 4.19. prohibit rule type
- 4.20. blackhole rule type
- 4.21. ICMP Redirect on the Wire
- 5.1. Stateless NAT Packet Capture
- 5.2. Basic commands to create a stateless NAT
- 5.3. Conditional Stateless NAT (not performing NAT for a specified destination network)
- 5.4. Using an ipchains packet filter with stateless NAT
- 5.5. Using DNAT for all protocols (and ports) on one IP
- 5.6. Using DNAT for a single port
- 5.7. Simulating full NAT with SNAT and DNAT
- 7.1. Blocking a destination and using the
`REJECT`

target, cf. Example D.17, “Adding a`prohibit`

route with route add” - 10.1. Multiple Outbound Internet links, part I; ip route
- 10.2. Multiple Outbound Internet links, part II; iptables
- 10.3. Multiple Outbound Internet links, part III; ip rule
- 10.4. Multiple Internet links, inbound traffic; using iproute2 only
- 11.1. Proxy ARP SysV initialization script
- 11.2. Proxy ARP configuration file
- 11.3. Static NAT SysV initialization script
- 11.4. Static NAT configuration file
- B.1. Displaying the arp table with arp
- B.2. Adding arp table entries with arp
- B.3. Deleting arp table entries with arp
- B.4. Displaying reachability of an IP on the local Ethernet with arping
- B.5. Duplicate Address Detection with arping
- B.6. Using ip link show
- B.7. Using ip link set to change device flags
- B.8. Deactivating a link layer device with ip link set
- B.9. Activating a link layer device with ip link set
- B.10. Using ip link set to change device flags
- B.11. Changing the device name with ip link set
- B.12. Changing broadcast and hardware addresses with ip link set
- B.13. Displaying the ARP cache with ip neighbor show
- B.14. Displaying the ARP cache on an interface with ip neighbor show
- B.15. Displaying the ARP cache for a particular network with ip neighbor show
- B.16. Entering a permanent entry into the ARP cache with ip neighbor add
- B.17. Entering a proxy ARP entry with ip neighbor add proxy
- B.18. Altering an entry in the ARP cache with ip neighbor change
- B.19. Removing an entry from the ARP cache with ip neighbor del
- B.20. Removing learned entries from the ARP cache with ip neighbor flush
- B.21. Detecting link layer status with mii-tool
- B.22. Specifying Ethernet port speeds with mii-tool --advertise
- B.23. Forcing Ethernet port speed with mii-tool --force
- C.1. Viewing interface information with ifconfig
- C.2. Bringing down an interface with ifconfig
- C.3. Bringing up an interface with ifconfig
- C.4. Changing MTU with ifconfig
- C.5. Setting interface flags with ifconfig
- C.6. Displaying IP information with ip address
- C.7. Adding IP addresses to an interface with ip address
- C.8. Removing IP addresses from interfaces with ip address
- C.9. Removing all IPs on an interface with ip address flush
- D.1. Viewing a simple routing table with route
- D.2. Viewing a complex routing table with route
- D.3. Viewing the routing cache with route
- D.4. Adding a static route to a network route add
- D.5. Adding a static route to a host with route add
- D.6. Adding a static route to a host on the same media with route add
- D.7. Setting the default route with route
- D.8. An alternate method of setting the default route with route
- D.9. Removing a static host route with route del
- D.10. Removing the default route with route del
- D.11. Viewing the main routing table with ip route show
- D.12. Viewing the local routing table with ip route show table local
- D.13. Viewing a routing table with ip route show table
- D.14. Displaying the routing cache with ip route show cache
- D.15. Displaying statistics from the routing cache with ip -s route show cache
- D.16. Adding a static route to a network with route add, cf. Example D.4, “Adding a static route to a network route add”
- D.17. Adding a
`prohibit`

route with route add - D.18. Using
`from`

in a routing command with route add - D.19. Using
`src`

in a routing command with route add - D.20. Setting the default route with ip route add default
- D.21. Creating a NAT route for a single IP with ip route add nat
- D.22. Creating a NAT route for an entire network with ip route add nat
- D.23. Removing routes with ip route del
- D.24. Altering existing routes with ip route change
- D.25. Testing routing tables with ip route get
- D.26. Removing a specific route and emptying a routing table with ip route flush
- D.27. Emptying the routing cache with ip route flush cache
- D.28. Displaying the RPDB with ip rule show
- D.29. Creating a simple entry in the RPDB with ip rule add
- D.30. Creating a complex entry in the RPDB with ip rule add
- D.31. Creating a NAT rule with ip rule add nat
- D.32. Creating a NAT rule for an entire network with ip rule add nat
- D.33. Removing a NAT rule for an entire network with ip rule del nat
- F.1. Simple use of nc
- F.2. Specifying timeout with nc
- F.3. Specifying source address with nc
- F.4. Using nc as a server
- F.5. Delaying a stream with nc
- F.6. Using nc with UDP
- F.7. Simple use of socat
- F.8. Using socat with proxy connect
- F.9. Using socat perform SSL
- F.10. Connecting one end of socat to a file descriptor
- F.11. Connecting socat to a serial line
- F.12. Using a PTY with socat
- F.13. Executing a command with socat
- F.14. Connecting one socat to another one
- F.15. Simple use of tcpclient
- F.16. Specifying the local port which tcpclient should request
- F.17. Specifying the local IP to which tcpclient should bind
- F.18. IP redirection with xinetd
- F.19. Publishing a service with xinetd
- F.20. Simple use of tcpserver
- F.21. Specifying a CDB for tcpserver
- F.22. Limiting the number of concurrently accept TCP sessions under tcpserver
- F.23. Specifying a UID for tcpserver's spawned processes
- F.24. Redirecting a TCP port with redir
- F.25. Running redir in transparent mode
- F.26. Running redir from another TCP server
- F.27. Specifying a source address for redir's client side
- G.1. Using ping to test reachability
- G.2. Using ping to specify number of packets to send
- G.3. Using ping to specify number of packets to send
- G.4. Using ping to stress a network
- G.5. Using ping to stress a network with large packets
- G.6. Recording a network route with ping
- G.7. Setting the TTL on a ping packet
- G.8. Setting ToS for a diagnostic ping
- G.9. Specifying a source address for ping
- G.10. Simple usage of traceroute
- G.11. Displaying IP socket status with netstat
- G.12. Displaying IP socket status details with netstat
- G.13. Displaying the main routing table with netstat
- G.14. Displaying the routing cache with netstat
- G.15. Displaying the masquerading table with netstat
- G.16. Viewing an ARP broadcast request and reply with tcpdump
- G.17. Viewing a gratuitous ARP packet with tcpdump
- G.18. Viewing unicast ARP packets with tcpdump
- G.19. tcpdump reporting port unreachable
- G.20. tcpdump reporting host unreachable
- G.21. tcpdump reporting net unreachable
- G.22. Monitoring TCP window sizes with tcpdump
- G.23. Examining TCP flags with tcpdump
- G.24. Examining TCP acknowledgement numbers with tcpdump
- G.25. Writing tcpdump data to a file
- G.26. Reading tcpdump data from a file
- G.27. Causing tcpdump to use a line buffer
- G.28. Understanding fragmentation as reported by tcpdump
- G.29. Specifying interface with tcpdump
- G.30. Timestamp related options to tcpdump

**Table of Contents**

This guide is as an overview of the IP networking capabilities of linux kernels 2.2 and 2.4. The target audience is any beginning to advanced network administrator who wants practical examples and explanation of rumoured features of linux. As the Internet is lousy with documentation on the nooks and crannies of linux networking support, I have tried to provide links to existing documentation on IP networking with linux.

The documentation you'll find here covers kernels 2.2 and 2.4, although a good number of the examples and concepts may also apply to older kernels. In the event that I cover a feature that is only present or supported under a particular kernel, I'll identify which kernel supports that feature.

I assume a few things about the reader. First, the reader has a basic understanding (at least) of IP addressing and networking. If this is not the case, or the reader has some trouble following my networking examples, I have provided a section of links to IP layer tutorials and general introductory documentation in the appendix. Second, I assume the reader is comfortable with command line tools and the Linux, Unix, or BSD environments. Finally, I assume the reader has working network cards and a Linux OS. For assistance with Ethernet cards, the there exists a good Ethernet HOWTO.

The examples I give are intended as tutorial examples only. The user should understand and accept the ramifications of using these examples on his/her own machines. I recommend that before running any example on a production machine, the user test in a controlled environment. I accept no responsibility for damage, misconfiguration or loss of any kind as a result of referring to this documentation. Proceed with caution at your own risk.

This guide has been written primarily as a companion reference to IP networking on Ethernets. Although I do allude to other link layer types occasionally in this book, the focus has been IP as used in Ethernet. Ethernet is one of the most common networking devices supported under linux, and is practically ubiquitous.

This text was written in
DocBook with
**vim**.
All formatting has been applied by
xsltproc based on
DocBook
and
LDP XSL
stylesheets.
Typeface formatting and display
conventions are similar to most printed and electronically distributed
technical documentation. A brief summary of these conventions
follows below.

The interactive shell prompt will look like

`[root@hostname]# `


for the root user and

`[user@hostname]$ `


for non-root users, although most of the operations we will be discussing will require root privileges.

Any commands to be entered by the user will always appear like

`{ echo "Hi, I am exiting with a non-zero exit code."; exit 1 }`


Output by any program will look something like this:

`Hi, I am exiting with a non-zero exit code.`


Where possible, an additional convention I have used is the suppression of all hostname lookup. DNS and other naming based schemes often confuse the novice and expert alike, particularly when the name resolver is slow or unreachable. Since the focus of this guide is IP layer networking, DNS names will be used only where absolutely unambiguous.

Perhaps this should be called things that are wrong with this document,
or things which should be improved. See the
`src/ROADMAP`

for notes on what is likely to be
forthcoming in subsequent releases.

The internal document linking, while good, but could be better. Especially lame is the lack of an index. External links should be used more commonly where appropriate instead of sending users to the links page.

If you are looking for LARTC topics, you may find some LAR topics
here, but you should try the LARTC
page itself if you have questions that are more TC than LAR.
Consult Appendix I, *Links to other Resources* for further references to available
documentation.

There are many tools available under linux which are also available under
other unix-like operating systems, but there are additional tools and
specific tools which are available only to users of linux. This guide
represents an effort to identify some of these tools. The most concrete
example of the difference between linux only tools and generally available
unix-like tools is the difference between the traditional
**ifconfig** and **route** commands,
available under most variants of unix, and the **iproute2**
command suite, written specificially for linux.

Because this guide concerns itself with the features, strengths, and
peculiarities of IP networking with linux, the **iproute2**
command suite assumes a prominent role. The **iproute2**
tools expose the strength, flexibility and potential of the linux
networking stack.

Many of the tools introduced and concepts introduced are also detailed in other HOWTOs and guides available at The Linux Documentation Project in addition to many other places on the Internet and in printed books.

As with many human endeavours, this work is made possible by the efforts of others. For me, this effort represents almost four years of learning and network administration. The knowledge collected here is in large measure a repackaging of disparate resources and my own experiences over time. Without the greater linux community, I would not be able to provide this resource.

I would like to take this opportunity to make a plug for my employer, SecurePipe, Inc. which has provided me stable and challenging employment for these (almost) four years. SecurePipe is a managed security services provider specializing in managed firewall, VPN, and IDS services to small and medium sized companies. They offer me the opportunity to hone my networking skills and explore areas of linux networking unknown to me. Thanks also to SecurePipe, Inc. for hosting this cost-free on their servers.

Over the course of the project, many people have contributed suggestions,
modifications, corrections and additions. I'll acknowledge them briefly
here. For full acknowledgements, see
`src/ACKNOWLEDGEMENTS`

in the DocBook source tree.

Russ Herrold,

*2002-09-22*Yann Hirou,

*2002-09-26*Julian Anastasov,

*2002-10-29*Bert Hubert,

*2002-11-14*Tony Kapela,

*2002-11-30*George Georgalis,

*2003-01-11*Alex Russell,

*2003-02-02*giovanni,

*2003-02-06*Gilles Douillet,

*2003-02-28*

Please feel free to point out any irregularities, factual errors,
typographical errors, or logical gaps in this
documentation. If you have rants or raves about this documentation,
please mail me directly at `<mabrown@securepipe.com>`

.

Now, let's begin! Let me welcome you to the pleasure and reliability of IP networking with linux.

**Table of Contents**

- 1. Basic IP Connectivity
- 2. Ethernet
- 3. Bridging
- 4. IP Routing
- 5. Network Address Translation (NAT)
- 5.1. Rationale for and Introduction to NAT
- 5.2. Application Layer Protocols with Embedded Network Information
- 5.3. Stateless NAT with
**iproute2** - 5.4. Stateless NAT and Packet Filtering
- 5.5. Destination NAT with netfilter (DNAT)
- 5.6. Port Address Translation (PAT) from Userspace
- 5.7. Transparent PAT from Userspace

- 6. Masquerading and Source Network Address Translation
- 7. Packet Filtering
- 8. Statefulness and Statelessness

**Table of Contents**

Internet Protocol (IP) networking is now among the most common networking technologies in use today. The IP stack under linux is mature, robust and reliable. This chapter covers the basics of configuring a linux machine or multiple linux machines to join an IP network.

This chapter covers a quick overview of the locations of the networking control files on different distributions of linux. The remainder of the chapter is devoted to outlining the basics of IP networking with linux.

These basics are written in a more tutorial style than the remainder of the first part of the book. Reading and understanding IP addressing and routing information is a key skill to master when beginning with linux. Naturally, the next step is to alter the IP configuration of a machine. This chapter will introduce these two key skills in a tutorial style. Subsequent chapters will engage specific subtopics of linux networking in a more thorough and less tutorial manner.

Different linux distribution vendors put their networking configuration files in different places in the filesystem. Here is a brief summary of the locations of the IP networking configuration information under a few common linux distributions along with links to further documentation.

**Location of networking configuration files**

RedHat (and Mandrake)

Interface definitions

`/etc/sysconfig/network-scripts/ifcfg-*`

Hostname and default gateway definition

`/etc/sysconfig/network`

Definition of static routes

`/etc/sysconfig/static-routes`


SuSe (version >= 8.0)

Interface definitions

`/etc/sysconfig/network/ifcfg-*`

Static route definition

`/etc/sysconfig/network/routes`

Interface specific static route definition

`/etc/sysconfig/network/ifroute-*`


SuSe (version <= 8.0)

Interface and route definitions

`/etc/rc.config`


Debian

Interface and route definitions

`/etc/network/interfaces`


Gentoo

Interface and route definitions

`/etc/conf.d/net`


Slackware

Interface and route definitions

`/etc/rc.d/rc.inet1`



The format of the networking configuration files differs significantly from distribution to distribution, yet the tools used by these scripts are the same. This documentation will focus on these tools and how they instruct the kernel to alter interface and route information. Consult the distribution's documentation for questions of file format and order of operation.

For the remainder of this document, many examples refer to machines in a hypothetical network. Refer to the example network description for the network map and addressing scheme.

Assuming an already configured machine named `tristan`

, let's
look at the IP addressing and routing
table. Next we'll examine how the machine
communicates with computers (hosts) on the locally reachable network. We'll
then send packets through our
default gateway to other networks. After learning what a default
route is, we'll look at a static
route.

One of the first things to learn about a machine attached to an IP
network is its IP address. We'll begin by looking at
a machine named `tristan`

on the main desktop network (192.168.99.0/24).

The machine `tristan`

is alive on IP 192.168.99.35 and
has been properly configured by the system administrator.
By examining the
**route**
and **ifconfig**
output we can learn a good deal about the network to which
`tristan`

is connected
[1].

**Example 1.1. Sample ifconfig output**

|

For the moment, ignore the loopback interface (lo) and concentrate
on the Ethernet interface. Examine the output of the
**ifconfig** command. We can learn a great deal about
the IP network to which we are connected simply by reading the
**ifconfig** output. For a thorough discussion of
**ifconfig**, see
Section C.1, “**ifconfig**”.

The IP address active on `tristan`

is 192.168.99.35. This means that
any IP packets created by `tristan`

will have a
source address of 192.168.99.35. Similarly any packet received by
`tristan`

will have the destination address of 192.168.99.35.
When creating an outbound packet `tristan`

will set the destination
address to the server's IP. This gives the remote host and the
networking devices in between these hosts enough information to
carry packets between the two devices.

Because `tristan`

will
advertise that it accepts packets with a destination address of
192.168.99.35, any frames (packets) appearing on the Ethernet
bound for 192.168.99.35 will reach `tristan`

. The process of
communicating the ownership of an IP address is called ARP. Read
Section 2.1.1, “Overview of Address Resolution Protocol” for a complete discussion of
this process.

This is fundamental to IP networking. It is fundamental that a host be able to generate and receive packets on an IP address assigned to it. This IP ad

[... 内容超长，已截断；完整原文见 source URL ...]
