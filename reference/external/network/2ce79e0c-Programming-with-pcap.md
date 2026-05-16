---
title: Programming with pcap
source: https://www.tcpdump.org/pcap.html
kind: external
domain: network
original_date: 1999-01-01
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.tcpdump.org](https://www.tcpdump.org/pcap.html)
> 原始日期：1999-01-01
> 抓取日期：2026-05-16

# Programming with pcap

At this point we have learned how to define a device,
prepare it for sniffing, and apply filters about what we should and should not
sniff for. Now it is time to actually capture some packets.

There are two main techniques for capturing packets. We can either
capture a single packet at a time, or we can enter a loop that waits for
*n* number of packets to be sniffed before being done. We will
begin by looking at how to capture a single packet, then look at methods
of using loops. For this we use
**pcap_next**(3PCAP).

The prototype is fairly simple:

u_char *pcap_next(pcap_t *p, struct pcap_pkthdr *h)

The first argument is our session handler. The second argument is a
pointer to a structure that holds general information about the packet,
specifically the time in which it was sniffed, the length of this packet, and
the length of this specific portion (in case it is fragmented, for example).
**pcap_next**()
returns a `u_char`

pointer to the packet that is described by this
structure. We'll discuss the technique for actually reading the packet
itself later.

Here is a simple demonstration of using **pcap_next**() to sniff a packet.

#include <pcap.h>
#include <stdio.h>
int main(int argc, char *argv[])
{
pcap_t *handle; /* Session handle */
char *dev; /* The device to sniff on */
char errbuf[PCAP_ERRBUF_SIZE]; /* Error string */
struct bpf_program fp; /* The compiled filter */
char filter_exp[] = "port 23"; /* The filter expression */
bpf_u_int32 mask; /* Our netmask */
bpf_u_int32 net; /* Our IP */
struct pcap_pkthdr header; /* The header that pcap gives us */
const u_char *packet; /* The actual packet */
/* Define the device */
dev = pcap_lookupdev(errbuf);
if (dev == NULL) {
fprintf(stderr, "Couldn't find default device: %s\n", errbuf);
return(2);
}
/* Find the properties for the device */
if (pcap_lookupnet(dev, &net, &mask, errbuf) == -1) {
fprintf(stderr, "Couldn't get netmask for device %s: %s\n", dev, errbuf);
net = 0;
mask = 0;
}
/* Open the session in promiscuous mode */
handle = pcap_open_live(dev, BUFSIZ, 1, 1000, errbuf);
if (handle == NULL) {
fprintf(stderr, "Couldn't open device %s: %s\n", dev, errbuf);
return(2);
}
/* Compile and apply the filter */
if (pcap_compile(handle, &fp, filter_exp, 0, net) == -1) {
fprintf(stderr, "Couldn't parse filter %s: %s\n", filter_exp, pcap_geterr(handle));
return(2);
}
if (pcap_setfilter(handle, &fp) == -1) {
fprintf(stderr, "Couldn't install filter %s: %s\n", filter_exp, pcap_geterr(handle));
return(2);
}
/* Grab a packet */
packet = pcap_next(handle, &header);
/* Print its length */
printf("Jacked a packet with length of [%d]\n", header.len);
/* And close the session */
pcap_close(handle);
return(0);
}

This application sniffs on whatever device is returned by **pcap_lookupdev**()
by
putting it into promiscuous mode. It finds the first packet to come across
port 23 (telnet) and tells the user the size of the packet (in bytes).
Again, this program includes a new call,
**pcap_close**(3PCAP),
which we will discuss
later (although it really is quite self explanatory).

The other technique we can use is more complicated, and
probably more useful. Few sniffers (if any) actually use **pcap_next**().
More often than not, they use
**pcap_loop**(3PCAP)
or
**pcap_dispatch**(3PCAP)
(which then themselves use **pcap_loop**()).
To understand the use of these two functions,
you must understand the idea of a callback function.

Callback functions are not anything new, and are very common in many
APIs. The concept behind a callback function is fairly simple.
Suppose I have a program that is waiting for an event of some sort. For
the purpose of this example, let's pretend that my program wants a user
to press a key on the keyboard. Every time they press a key, I want to
call a function which then will determine that to do. The function I am
utilizing is a callback function. Every time the user presses a key, my
program will call the callback function. Callbacks are used in pcap,
but instead of being called when a user presses a key, they are called
when pcap sniffs a packet. The two functions that one can use to define
their callback are **pcap_loop**() and **pcap_dispatch**(),
these are very similar in their usage of callbacks. Both of
them call a callback function every time a packet is sniffed that meets
our filter requirements (if any filter exists, of course. If not, then
*all* packets that are sniffed are sent to the callback.)

The prototype for **pcap_loop**() is below:

int pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)

The first argument is our session handle. Following that is an
integer that tells **pcap_loop**() how many packets it should sniff for
before returning (a negative value means it should sniff until an error
occurs). The third argument is the name of the callback function (just
its identifier, no parentheses). The last argument is useful in some
applications, but many times is simply set as `NULL`

. Suppose we have
arguments of our own that we wish to send to our callback function, in
addition to the arguments that **pcap_loop**() sends. This is where we do
it. Obviously, you must typecast to a `u_char`

pointer to ensure the
results make it there correctly; as we will see later, pcap makes use of
some very interesting means of passing information in the form of a
`u_char`

pointer. After we show an example of how pcap does it, it should
be obvious how to do it here. If not, consult your local C reference
text, as an explanation of pointers is beyond the scope of this
document. **pcap_dispatch**() is almost identical in usage. The only
difference between these two functions is that **pcap_dispatch**()
will only process the first batch of packets that it receives from the system, while
**pcap_loop**() will continue processing
packets or batches of packets until the count of packets runs out. For
a more in depth discussion of their differences, see the man page.

Before we can provide an example of using **pcap_loop**(),
we must examine the format of our callback function. We
cannot arbitrarily define our callback's prototype; otherwise, **pcap_loop**()
would not know how to use the function. So we use this format as the prototype
for our callback function:

void got_packet(u_char *args, const struct pcap_pkthdr *header,
const u_char *packet);

Let's examine this in more detail. First, you'll notice that the function
has a `void`

return type. This is logical, because **pcap_loop**()
wouldn't know how to handle a return value anyway. The first argument
corresponds to the last argument of **pcap_loop**().
Whatever value is passed as the last argument to **pcap_loop**()
is passed to the first argument of our callback function
every time the function is called. The second argument is the pcap header,
which contains information about when the packet was sniffed, how large it is,
etc. The `pcap_pkthdr`

structure is defined in `pcap.h`

as:

struct pcap_pkthdr {
struct timeval ts; /* time stamp */
bpf_u_int32 caplen; /* length of portion present */
bpf_u_int32 len; /* length this packet (off wire) */
};

These values should be fairly self explanatory. The last argument is
the most interesting of them all, and the most confusing to the average
novice pcap programmer. It is another pointer to a `u_char`

, and it
points to the first byte of a chunk of data containing the entire
packet, as sniffed by **pcap_loop**().

But how do you make use of this variable (named `packet`

in
our prototype)? A packet contains many attributes, so as you can
imagine, it is not really a string, but actually a collection of
structures (for instance, a TCP/IP packet would have an Ethernet header,
an IP header, a TCP header, and lastly, the packet's payload). This
`u_char`

pointer points to the serialized version of these structures. To
make any use of it, we must do some interesting typecasting.

First, we must have the actual structures
defined before we can typecast to them. The following are the structure
definitions that I use to describe a TCP/IP packet over Ethernet.

/* Ethernet addresses are 6 bytes */
#define ETHER_ADDR_LEN 6
/* Ethernet header */
struct sniff_ethernet {
u_char ether_dhost[ETHER_ADDR_LEN]; /* Destination host address */
u_char ether_shost[ETHER_ADDR_LEN]; /* Source host address */
u_short ether_type; /* IP? ARP? RARP? etc */
};
/* IP header */
struct sniff_ip {
u_char ip_vhl; /* version << 4 | header length >> 2 */
u_char ip_tos; /* type of service */
u_short ip_len; /* total length */
u_short ip_id; /* identification */
u_short ip_off; /* fragment offset field */
#define IP_RF 0x8000 /* reserved fragment flag */
#define IP_DF 0x4000 /* don't fragment flag */
#define IP_MF 0x2000 /* more fragments flag */
#define IP_OFFMASK 0x1fff /* mask for fragmenting bits */
u_char ip_ttl; /* time to live */
u_char ip_p; /* protocol */
u_short ip_sum; /* checksum */
struct in_addr ip_src,ip_dst; /* source and dest address */
};
#define IP_HL(ip) (((ip)->ip_vhl) & 0x0f)
#define IP_V(ip) (((ip)->ip_vhl) >> 4)
/* TCP header */
typedef u_int tcp_seq;
struct sniff_tcp {
u_short th_sport; /* source port */
u_short th_dport; /* destination port */
tcp_seq th_seq; /* sequence number */
tcp_seq th_ack; /* acknowledgement number */
u_char th_offx2; /* data offset, rsvd */
#define TH_OFF(th) (((th)->th_offx2 & 0xf0) >> 4)
u_char th_flags;
#define TH_FIN 0x01
#define TH_SYN 0x02
#define TH_RST 0x04
#define TH_PUSH 0x08
#define TH_ACK 0x10
#define TH_URG 0x20
#define TH_ECE 0x40
#define TH_CWR 0x80
#define TH_FLAGS (TH_FIN|TH_SYN|TH_RST|TH_ACK|TH_URG|TH_ECE|TH_CWR)
u_short th_win; /* window */
u_short th_sum; /* checksum */
u_short th_urp; /* urgent pointer */
};

So how does all of this relate to pcap and our mysterious `u_char`

pointer? Well, those structures define the headers that appear in the
data for the packet. So how can we break it apart? Be prepared to
witness one of the most practical uses of pointers (for all of those new
C programmers who insist that pointers are useless, I smite you).

Again, we're going to assume that we are dealing with a TCP/IP packet
over Ethernet. This same technique applies to any packet; the only
difference is the structure types that you actually use. So let's begin
by defining the variables and compile-time definitions we will need to
deconstruct the packet data.

/* ethernet headers are always exactly 14 bytes */
#define SIZE_ETHERNET 14
const struct sniff_ethernet *ethernet; /* The ethernet header */
const struct sniff_ip *ip; /* The IP header */
const struct sniff_tcp *tcp; /* The TCP header */
const char *payload; /* Packet payload */
u_int size_ip;
u_int size_tcp;


And now we do our magical typecasting:

ethernet = (struct sniff_ethernet*)(packet);
ip = (struct sniff_ip*)(packet + SIZE_ETHERNET);
size_ip = IP_HL(ip)*4;
if (size_ip < 20) {
printf(" * Invalid IP header length: %u bytes\n", size_ip);
return;
}
tcp = (struct sniff_tcp*)(packet + SIZE_ETHERNET + size_ip);
size_tcp = TH_OFF(tcp)*4;
if (size_tcp < 20) {
printf(" * Invalid TCP header length: %u bytes\n", size_tcp);
return;
}
payload = (u_char *)(packet + SIZE_ETHERNET + size_ip + size_tcp);

How does this work? Consider the layout of the packet data in memory.
The `u_char`

pointer is really just a variable containing an address in
memory. That's what a pointer is; it points to a location in memory.

For the sake of simplicity, we'll say that the address this pointer is
set to is the value X. Well, if our three structures are just sitting
in line, the first of them (`sniff_ethernet`

) being located in memory at
the address X, then we can easily find the address of the structure
after it; that address is X plus the length of the Ethernet header,
which is 14, or `SIZE_ETHERNET`

.

Similarly if we have the address of that header, the address of the
structure after it is the address of that header plus the length of that
header. The IP header, unlike the Ethernet header, does
**not** have a fixed length; its length is given, as a
count of 4-byte words, by the header length field of the IP header. As
it's a count of 4-byte words, it must be multiplied by 4 to give the
size in bytes. The minimum length of that header is 20 bytes.

The TCP header also has a variable length; its length is given, as a
number of 4-byte words, by the "data offset" field of the TCP header,
and its minimum length is also 20 bytes.

So let's make a chart:


| Variable |
Location (in bytes) |
`sniff_ethernet` |
X |
`sniff_ip` |
X + `SIZE_ETHERNET` |
`sniff_tcp` |
X + `SIZE_ETHERNET` + {IP header length} |
`payload` |
X + `SIZE_ETHERNET` + {IP header length} + {TCP header length} |

The `sniff_ethernet`

structure, being the first in line, is simply at
location X. `sniff_ip`

, who follows directly after `sniff_ethernet`

, is at
the location X, plus however much space the Ethernet header consumes (14
bytes, or `SIZE_ETHERNET`

). `sniff_tcp`

is after both `sniff_ip`

and
`sniff_ethernet`

, so it is location at X plus the sizes of the Ethernet
and IP headers (14 bytes, and 4 times the IP header length,
respectively). Lastly, the payload (which doesn't have a single
structure corresponding to it, as its contents depends on the protocol
being used atop TCP) is located after all of them.

So at this point, we know how to set our
callback function, call it, and find out the attributes about the packet that
has been sniffed. It's now the time you have been waiting for: writing a
useful packet sniffer. Because of the length of the source code, I'm not
going to include it in the body of this document. Simply download
`sniffex.c`

and try it out.