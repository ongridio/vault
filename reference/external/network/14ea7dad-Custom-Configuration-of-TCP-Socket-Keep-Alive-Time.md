---
title: Custom Configuration of TCP Socket Keep-Alive Timeouts
source: http://coryklein.com/tcp/2015/11/25/custom-configuration-of-tcp-socket-keep-alive-timeouts.html
kind: external
domain: network
author: Cory-Klein
original_date: 2015-11-25
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[coryklein.com](http://coryklein.com/tcp/2015/11/25/custom-configuration-of-tcp-socket-keep-alive-timeouts.html)
> 作者：Cory-Klein
> 原始日期：2015-11-25
> 抓取日期：2026-05-16

# Custom Configuration of TCP Socket Keep-Alive Timeouts

# Custom Configuration of TCP Socket Keep-Alive Timeouts

This post is a modified and improved version of an answer I recently posted on StackOverflow.

## Introduction

TCP connections consist of two sockets, one on each end of the connection. When one side wants to terminate the connection, it sends an `RST`

packet which the other side acknowledges and both close their sockets.

Until that happens, however, both sides will keep their socket open indefinitely. This leaves open the possibility that one side may close their socket, either intentionally or due to some error, without informing the other end via `RST`

. In order to detect this scenario and close stale connections the TCP Keep Alive process is used.

## Keep-Alive Process

There are three configurable properties that determine how Keep-Alives work. On Linux they are1:

`tcp_keepalive_time`

- default 7200 seconds

`tcp_keepalive_probes`

- default 9

`tcp_keepalive_intvl`

- default 75 seconds


The process works like this:

- Client opens TCP connection
- If the connection is silent for
`tcp_keepalive_time`

seconds, send a single empty`ACK`

packet.1 - Did the server respond with a corresponding
`ACK`

of its own?**No**- Wait
`tcp_keepalive_intvl`

seconds, then send another`ACK`

- Repeat until the number of
`ACK`

probes that have been sent equals`tcp_keepalive_probes`

. - If no response has been received at this point, send a
`RST`

and terminate the connection.

- Wait
**Yes**: Return to step 2


This process is enabled by default on most operating systems, and thus dead TCP connections are regularly pruned once the other end has been non-responsive for 2 hours 11 minutes (7200 seconds + 75 * 9 seconds).

## Gotchas

### 2 Hour Default

Since the process doesn’t start until a connection has been idle for two hours by default, stale TCP connections can linger for a very long time before being pruned. This can be especially harmful for expensive connections such as database connections.

### Keep-Alive is Optional

According to RFC 1122 4.2.3.6, responding to and/or relaying TCP Keep-Alive packets *is optional*:

Implementors MAY include “keep-alives” in their TCP implementations, although this practice is not universally accepted. If keep-alives are included, the application MUST be able to turn them on or off for each TCP connection, and they MUST default to off.


…


It is extremely important to remember that ACK segments that contain no data are not reliably transmitted by TCP.


The reasoning being that Keep-Alive packets contain no data and are not strictly necessary and risk clogging up the tubes of the interwebs if overused.

*In practice however*, my experience has been that this concern has dwindled over time as bandwidth has become cheaper and Keep-Alive packets are not dropped. Amazon EC2 documentation for instance gives an indirect endorsement of Keep-Alive, so if you’re hosting with AWS you are likely safe relying on Keep-Alive, but your mileage may vary.

## Changing TCP Timeouts

### Per Socket

#### C/C++

```
#include <sys/socket.h>
int keepalive_enabled = 1;
int keepalive_time = 180;
int keepalive_count = 3;
int keepalive_interval = 10;
setsockopt(socket_file_descriptor, SOL_SOCKET, SO_KEEPALIVE, 1,
setsockopt(socket_file_descriptor, IPPROTO_TCP, TCP_KEEPIDLE, keepalive_time, sizeof keepalive_time);
setsockopt(socket_file_descriptor, IPPROTO_TCP, TCP_KEEPCNT, keepalive_count, sizeof keepalive_count);
setsockopt(socket_file_descriptor, IPPROTO_TCP, TCP_KEEPINTVL, keepalive_interval, sizeof keepalive_interval);
```


#### Java

Unfortunately since TCP connections are managed on the OS level, Java does not support configuring timeouts on a per-socket level such as in `java.net.Socket`

. I have found some attempts3 to use Java Native Interface (JNI) to create Java sockets that call native code to configure these options, but none appear to have widespread community adoption or support.

### OS Level

In some cases (especially Java), you may be forced to apply your configuration to the operating system as a whole. Be aware that this configuration will affect all TCP connections running on the entire system.

#### Linux

The currently configured TCP Keep-Alive settings can be found in

`/proc/sys/net/ipv4/tcp_keepalive_time`

`/proc/sys/net/ipv4/tcp_keepalive_probes`

`/proc/sys/net/ipv4/tcp_keepalive_intvl`


You can update any of these like so:

```
# Send first Keep-Alive packet when a TCP socket has been idle for 3 minutes
$ echo 180 > /proc/sys/net/ipv4/tcp_keepalive_time
# Send three Keep-Alive probes...
$ echo 3 > /proc/sys/net/ipv4/tcp_keepalive_probes
# ... spaced 10 seconds apart.
$ echo 10 > /proc/sys/net/ipv4/tcp_keepalive_intvl
```


These changes will not persist through a restart. To make persistent changes, use `sysctl`

:

```
sysctl -w net.ipv4.tcp_keepalive_time=180 net.ipv4.tcp_keepalive_probes=3 net.ipv4.tcp_keepalive_intvl=10
```


#### Mac OS X

The currently configured settings can be viewed with `sysctl`

:

```
$ sysctl net.inet.tcp | grep -E "keepidle|keepintvl|keepcnt"
net.inet.tcp.keepidle: 7200000
net.inet.tcp.keepintvl: 75000
net.inet.tcp.keepcnt: 8
```


Of note, Mac OS X defines `keepidle`

and `keepintvl`

in terms of milliseconds as opposed to Linux using seconds.

The properties can be set with `sysctl`

which will persist these settings across reboots:

```
sysctl -w net.inet.tcp.keepidle=180000 net.inet.tcp.keepcnt=3 net.inet.tcp.keepintvl=10000
```


Alternatively, you can add them to `/etc/sysctl.conf`

(creating the file if it doesn’t exist).

```
$ cat /etc/sysctl.conf
net.inet.tcp.keepidle=180000
net.inet.tcp.keepintvl=10000
net.inet.tcp.keepcnt=3
```


#### Windows

I don’t have a Windows machine to confirm, but you should find the respective TCP Keep-Alive settings in the registry at

`\HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\TCPIP\Parameters`


See here for more information.

*Footnotes*

1. See man tcp for more information.

2. This packet is often referred to as a “Keep-Alive” packet, but within the TCP specification it is just a regular ACK packet. Applications like Wireshark are able to label it as a “Keep-Alive” packet by meta-analysis of the sequence and acknowledgement numbers it contains in reference to the preceding communications on the socket.

3. Some examples I found from a basic Google search are lucwilliams/JavaLinuxNet and flonatel/libdontdie.