---
title: Linux网络编程-tcp缓存设置
source: http://www.cnblogs.com/whuqin/p/5580895.html
kind: external
domain: network
author: 春文秋武
original_date: 2016-06-13
fetched_at: 2026-05-16
bookmark_title: Linux网络编程-tcp缓存设置 - 春文秋武 - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/whuqin/p/5580895.html)
> 作者：春文秋武
> 原始日期：2016-06-13
> 抓取日期：2026-05-16

# Linux网络编程-tcp缓存设置

# Linux网络编程-tcp缓存设置

最近发现服务的逻辑完成时间很短，但是上游接收到的时间比较长，所以就怀疑是底层数据的序列化/反序列化、读写、传输有问题，然后怀疑是TCP的读写缓存是不是设置太小。现在就记录下TCP缓存的各配置项以及缓存大小的计算公式。

### 1.有关发送、接收缓存的配置

内核设置的套接字缓存

/proc/sys/net/core/rmem_default，net.core.rmem_default，套接字接收缓存默认值 (bit)

/proc/sys/net/core/wmem_default，net.core.wmem_default，套接字发送缓存默认值 (bit)

/proc/sys/net/core/rmem_max，net.core.rmem_max，套接字接收缓存最大值 (bit)

/proc/sys/net/core/wmem_max，net.core.wmem_max，发送缓存最大值 (bit)

tcp缓存

/proc/sys/net/ipv4/tcp_rmem：net.ipv4.tcp_rmem，接收缓存设置，依次代表最小值、默认值和最大值(bit)

4096 87380 4194304

/proc/sys/net/ipv4/tcp_wmem：net.ipv4.tcp_wmem，发送缓存设置，依次代表最小值、默认值和最大值(bit)

/proc/sys/net/ipv4/tcp_mem:

94500000 915000000 927000000

对应net.ipv4.tcp_mem，tcp整体缓存设置，对所有tcp内存使用状况的控制，单位是页，依次代表TCP整体内存的无压力值、压力模式开启阀值、最大使用值，用于控制新缓存的分配是否成功

**tcp或者udp的设置会覆盖内核设置**。**其中只有tcp_mem是用于tcp整体内存的控制，其他都是针对单个连接的。**

**2.修改配置**

sysctl -w net.core.rmem_max=8388608 ... sysctl -w net.ipv4.tcp_mem='8388608 8388608 8388608'

### 3.估算TCP缓存大小

以tcp接收缓存为例（实际上发送窗口=对方的接收窗口），tcp接收缓存有2部分组成：接收窗口及应用缓存，应用缓存用于应用的延时读及一些调度信息。linux使用net.ipv4.tcp_adv_win_scale（对应文件/proc/sys/net/ipv4/tcp_adv_win_scale）指出应用缓存的比例。

if tcp_adv_win_scale > 0: 应用缓存 = buffer / (2^tcp_adv_win_scale)，tcp_adv_win_scale默认值为2，表示缓存的四分之一用于应用缓存，可用接收窗口占四分之三。

if tcp_adv_win_scale <= 0: 应用缓存 = buffer - buffer/2^(-tcp_adv_win_scale)，即接收窗口=buffer/2^(-tcp_adv_win_scale)，如果tcp_adv_win_scale=-2，接收窗口占接收缓存的四分之一。

那如果能估算出接收窗口就能算出套接字缓存的大小。如何算接收窗口呢？

BDP(bandwidth-delay product，带宽时延积) = bandwith(bits/sec) * delay(sec)，代表网络传输能力，为了充分利用网络，最大接收窗口应该等于BDP。delay = RTT/2。

receive_win = bandwith * RTT / 2 buffer = rec_win/(3/4) (上面知道tcp_adv_win_scale=2时表示接收窗口占buffer的3/4

以我们的机房为例，同机房的带宽为30Gbit/s，两台机器ping可获得RTT大概为0.1ms，那BDP=(30Gb/1000) * 0.1 / 2 = 1.5Mb，buffer = 1.5Mb * 4 / 3 = 2Mb

### 4.TCP缓存的综合考虑

如第三节我们真的能设置tcp最大缓存为2Mb吗？通常一台机器会部署多个服务，一个服务内部也往往会建立多个tcp连接。但系统内存是有限的，如果有4000个连接，满负荷工作，达到最大窗口。那么tcp整体消耗内存=4000 * 2Mb = 1GB。

**并发连接越多，默认套接字缓存越大，则tcp占用内存越大。当套接字缓存和系统内存一定时，会影响并发连接数。对于高并发连接场景，系统资源不足，缩小缓存限制；并发连接少时，可以适当放大缓存限制。**

linux自身引入了自动调整接收缓存的功能，来使吞吐量最大，(缓存最大值还是受限于tcp_rmem[2])。配置项如下。

net.ipv4.tcp_moderate_rcvbuf = 1 (/proc/sys/net/ipv4/tcp_moderate_rcvbuf)