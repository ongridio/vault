---
title: Linux网络协议栈(四)——链路层(1)
source: https://www.cnblogs.com/hustcat/archive/2009/09/26/1574371.html
kind: external
domain: network
author: YY哥
original_date: 2009-09-26
fetched_at: 2026-05-16
bookmark_title: Linux网络协议栈(四)——链路层(1) - YY哥 - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/hustcat/archive/2009/09/26/1574371.html)
> 作者：YY哥
> 原始日期：2009-09-26
> 抓取日期：2026-05-16

# Linux网络协议栈(四)——链路层(1)

# Linux网络协议栈(四)——链路层(1)

1、接收帧

当网络适配器接收到数据帧时，就会触发一个中断，中断处理程序执行一些需要及时处理的任务，然后在下半部进行其它可以延迟的处理。中断处理程序主要进行以下一些操作：

(1) 分配sk_buff数据结构，并将接收到的数据帧从网络适配器I/O端口拷贝到sk_buff缓冲区中；

(2) 从数据帧中提取出一些信息，并设置sk_buff相应的参数，这些参数将被上层的网络协议使用，例如skb->protocol；

(3) 通过软中断NET_RX_SOFTIRQ通知内核接收到新的数据帧。


内核2.5中引入一组新的API来处理接收的数据帧，即NAPI。所以，当网络适配器接收到数据帧时，驱动有两种方式通知内核：(1)通过以前的函数netif_rx；(2)通过NAPI机制，但是只有很少的驱动使用它。

1.1、softnet_data数据结构



throttle、avg_blog和cng_level

这三个参数主要用于阻塞管理(congestion management)。throttle非0时，表示CPU负荷过载，它的值取决于input_pkt_queue，当throttle设置时，CPU接收到的所有数据帧都被丢弃。avg_blog表示输入队列input_pkt_queue的平均长度，它的范围从0到netdev_max_backlog(最大值)，它用来计算cng_level。cng_level表示阻塞的程度. avg_blog 和 cng_level与CPU处理的input_pkt_queue相关联,仅用于non_NAPI设备。


struct net_device *output_queue;

struct sk_buff *completion_queue;

这两个域用于发送数据。


struct sk_buff_head input_pkt_queue;

struct list_head poll_list;

struct net_device backlog_dev;

这三个域用于接收数据，其中input_pkt_queue与backlog_dev仅用于non-NAPI的 NIC，input_pkt_queue是接收到的数据队列头，它用于netif_rx()中，并最终由虚拟的poll函数 process_backlog()处理这个SKB队列。

poll_list则是有数据包等待处理的NIC设备队列。对于non-NAPI驱动来说，它始终是backlog_dev。


Softnet_data的初始化：

每个CPU的softnet_data是在net_dev_init中初始化的，代码如下：


来看看vortex_rx是怎么调用netif_rx的，大部分的网络设备驱动使用方式与其相似：


netif_rx( )函数

netif_rx通常在驱动的中断处理程序(严格意义来说，应该是中断服务例程，ISR)中被调用，但是也有例外，那就是回环设备。



1.3、NAPI方式

net_device中与NAPI相关的字段：

poll

从设备的输入队列取缓冲区的虚拟函数。对于NAPI，输入队列是私有的；对于NON_API设备，输入队列为softnet_data->input_pkt_queue。

poll_list

输入队列有新的数据帧需要处理的设备的链表，这些设备此时处于polling状态，表头为softnet_data->poll_list。位于链表中的设备关闭中断，正被内核查询。

quota

weight

quota表示poll每次可以从输入队列中取的最大缓冲区数。weight 成员描述接口的相对重要性：当资源紧张时，有多少流量可以从接口收到。如何设置 weight 参数没有严格的规则；依照惯例, 10 MBps 以太网接口设置 weight 为 16, 而快一些的接口使用 64. 你不能设置 weight 为一个超过你的接口能够存储的报文数目的值。quota值以weight为基础进行更新。


使用NAPI：



netif_rx_schedule函数




1.4、net_rx_action( )函数

net_rx_action为处理接收数据帧的下半部函数，输入的数据帧在两个地方等待net_rx_action来处理：

(1) CPU的输入队列。这是针对NON-NAPI方式的，它调用netif_rx，将数据帧加入到CPU的输入队列softnet_data->input_pkt_queue。

(2) 设备缓存。对于NAPI方式，poll函数从设备缓存读取数据帧。


1.5、process_backlog函数

对于non-API方式，poll函数为process_backlog：



1.6 NAPI的poll函数

这种方式下，NIC驱动程序会提供自己的poll函数和私有接收队列。

如intel 8255x系列网卡程序e100,它有在初始化的时候首先分配一个接收队列，而不像以上那种方式在接收到数据帧的时候再为其分配数据空间。这样，NAPI的poll函数在处理接收的时候，它遍历的是自己的私有队列：



1.7 netif_receive_skb函数

netif_receive_skb是链路层接收数据报的最后一站。它根据注册在全局数组ptype_all和ptype_base里的网络层数据报类型，把数据报递交给不同的网络层协议的接收函数(INET域中主要是ip_rcv和arp_rcv)。


当网络适配器接收到数据帧时，就会触发一个中断，中断处理程序执行一些需要及时处理的任务，然后在下半部进行其它可以延迟的处理。中断处理程序主要进行以下一些操作：

(1) 分配sk_buff数据结构，并将接收到的数据帧从网络适配器I/O端口拷贝到sk_buff缓冲区中；

(2) 从数据帧中提取出一些信息，并设置sk_buff相应的参数，这些参数将被上层的网络协议使用，例如skb->protocol；

(3) 通过软中断NET_RX_SOFTIRQ通知内核接收到新的数据帧。

内核2.5中引入一组新的API来处理接收的数据帧，即NAPI。所以，当网络适配器接收到数据帧时，驱动有两种方式通知内核：(1)通过以前的函数netif_rx；(2)通过NAPI机制，但是只有很少的驱动使用它。

1.1、softnet_data数据结构

//include/linux/netdevice.h

struct softnet_data

{

int throttle;

int cng_level;

int avg_blog;

struct sk_buff_head input_pkt_queue;

struct list_head poll_list;

struct net_device *output_queue;

struct sk_buff *completion_queue;


struct net_device backlog_dev; /* Sorry. 8) */

};


这个数据结构同时用于接收与发送数据包，它为per_CPU结构，这样每个CPU有自己独立的信息，这样在SMP之间就避免了加锁操作，从而大大提高了数据处理的并发性。struct softnet_data

{

int throttle;

int cng_level;

int avg_blog;

struct sk_buff_head input_pkt_queue;

struct list_head poll_list;

struct net_device *output_queue;

struct sk_buff *completion_queue;

struct net_device backlog_dev; /* Sorry. 8) */

};

throttle、avg_blog和cng_level

这三个参数主要用于阻塞管理(congestion management)。throttle非0时，表示CPU负荷过载，它的值取决于input_pkt_queue，当throttle设置时，CPU接收到的所有数据帧都被丢弃。avg_blog表示输入队列input_pkt_queue的平均长度，它的范围从0到netdev_max_backlog(最大值)，它用来计算cng_level。cng_level表示阻塞的程度. avg_blog 和 cng_level与CPU处理的input_pkt_queue相关联,仅用于non_NAPI设备。

struct net_device *output_queue;

struct sk_buff *completion_queue;

这两个域用于发送数据。

struct sk_buff_head input_pkt_queue;

struct list_head poll_list;

struct net_device backlog_dev;

这三个域用于接收数据，其中input_pkt_queue与backlog_dev仅用于non-NAPI的 NIC，input_pkt_queue是接收到的数据队列头，它用于netif_rx()中，并最终由虚拟的poll函数 process_backlog()处理这个SKB队列。

poll_list则是有数据包等待处理的NIC设备队列。对于non-NAPI驱动来说，它始终是backlog_dev。

Softnet_data的初始化：

每个CPU的softnet_data是在net_dev_init中初始化的，代码如下：

for (i = 0; i < NR_CPUS; i++) {

struct softnet_data *queue;


queue = &per_cpu(softnet_data,i);

skb_queue_head_init(&queue->input_pkt_queue);

queue->throttle = 0;

queue->cng_level = 0;

queue->avg_blog = 10; /* arbitrary non-zero */

queue->completion_queue = NULL;

INIT_LIST_HEAD(&queue->poll_list);

set_bit(_ _LINK_STATE_START, &queue->backlog_dev.state);

queue->backlog_dev.weight = weight_p;

queue->backlog_dev.poll = process_backlog;

atomic_set(&queue->backlog_dev.refcnt, 1);

}


1.2、NON-NAPI方式：struct softnet_data *queue;

queue = &per_cpu(softnet_data,i);

skb_queue_head_init(&queue->input_pkt_queue);

queue->throttle = 0;

queue->cng_level = 0;

queue->avg_blog = 10; /* arbitrary non-zero */

queue->completion_queue = NULL;

INIT_LIST_HEAD(&queue->poll_list);

set_bit(_ _LINK_STATE_START, &queue->backlog_dev.state);

queue->backlog_dev.weight = weight_p;

queue->backlog_dev.poll = process_backlog;

atomic_set(&queue->backlog_dev.refcnt, 1);

}

来看看vortex_rx是怎么调用netif_rx的，大部分的网络设备驱动使用方式与其相似：

static int vortex_rx(struct net_device *dev)

{

//…

struct sk_buff *skb;

skb = dev_alloc_skb(pkt_len + 5);//分配缓冲区

if (skb != NULL) {

skb->dev = dev;//设置接收包的网络设备

//将data和tail指针下移2个字节,使得IP头在缓冲区存储时可以16字节的边界上对齐

skb_reserve(skb, 2); /* Align IP on 16 byte boundaries */

//将数据帧从I/O端口拷贝到sk_buff缓冲区

skb->protocol = eth_type_trans(skb, dev);

netif_rx(skb);

dev->last_rx = jiffies;//接收到数据的时间

vp->stats.rx_packets++;

//…

}

}


{

//…

struct sk_buff *skb;

skb = dev_alloc_skb(pkt_len + 5);//分配缓冲区

if (skb != NULL) {

skb->dev = dev;//设置接收包的网络设备

//将data和tail指针下移2个字节,使得IP头在缓冲区存储时可以16字节的边界上对齐

skb_reserve(skb, 2); /* Align IP on 16 byte boundaries */

//将数据帧从I/O端口拷贝到sk_buff缓冲区

skb->protocol = eth_type_trans(skb, dev);

netif_rx(skb);

dev->last_rx = jiffies;//接收到数据的时间

vp->stats.rx_packets++;

//…

}

}

netif_rx( )函数

netif_rx通常在驱动的中断处理程序(严格意义来说，应该是中断服务例程，ISR)中被调用，但是也有例外，那就是回环设备。

Code

这段代码关键是，将这个SKB加入到相应的input_pkt_queue队列中，并调用netif_rx_schedule()，而对于NAPI方式，它没有使用input_pkt_queue队列，而是使用私有的队列，所以它没有这一个步骤。1.3、NAPI方式

net_device中与NAPI相关的字段：

poll

从设备的输入队列取缓冲区的虚拟函数。对于NAPI，输入队列是私有的；对于NON_API设备，输入队列为softnet_data->input_pkt_queue。

poll_list

输入队列有新的数据帧需要处理的设备的链表，这些设备此时处于polling状态，表头为softnet_data->poll_list。位于链表中的设备关闭中断，正被内核查询。

quota

weight

quota表示poll每次可以从输入队列中取的最大缓冲区数。weight 成员描述接口的相对重要性：当资源紧张时，有多少流量可以从接口收到。如何设置 weight 参数没有严格的规则；依照惯例, 10 MBps 以太网接口设置 weight 为 16, 而快一些的接口使用 64. 你不能设置 weight 为一个超过你的接口能够存储的报文数目的值。quota值以weight为基础进行更新。

使用NAPI：

Code

可以看到，两种方式的不同之处在于，NAPI方式直接调用 netif_rx_schedule，而非NAPI方式则要通过辅助函数netif_rx()设置好接收队列再调用 netif_rx_schedule()，再者，在非NAPI方式中，提交的是 netif_rx_schedule(&queue->backlog_dev)，而NAPI中，提交的是 netif_rx_schedule (netdev)，即是设备驱动的net_device结构，而不是queue中的backlog_dev。netif_rx_schedule函数

Code

整个过程如下：1.4、net_rx_action( )函数

net_rx_action为处理接收数据帧的下半部函数，输入的数据帧在两个地方等待net_rx_action来处理：

(1) CPU的输入队列。这是针对NON-NAPI方式的，它调用netif_rx，将数据帧加入到CPU的输入队列softnet_data->input_pkt_queue。

(2) 设备缓存。对于NAPI方式，poll函数从设备缓存读取数据帧。

Code

1.5、process_backlog函数

对于non-API方式，poll函数为process_backlog：

Code

该函数主要从CPU输入队列中取出套接字缓冲区，然后通过调用netif_receive_skb，将sb_buff传递给上层协议处理。budget 参数提供了一个我们允许传给内核的最大报文数目。在设备结构里, quota 成员给出了另一个最大值； poll 方法必须遵守这两个限制中的较小者。它也应当以实际收到的报文数目递减 dev->quota 和 *budget. budget 值是当前 CPU 能够从所有接口收到的最多报文数目, 而 quota 是一个每接口值, 常常在初始化时安排给接口以 weight 为起始。1.6 NAPI的poll函数

这种方式下，NIC驱动程序会提供自己的poll函数和私有接收队列。

如intel 8255x系列网卡程序e100,它有在初始化的时候首先分配一个接收队列，而不像以上那种方式在接收到数据帧的时候再为其分配数据空间。这样，NAPI的poll函数在处理接收的时候，它遍历的是自己的私有队列：

Code

主要工作在e100_rx_indicate()中完成，这主要重设SKB的一些参数，然后跟process_backlog(),一样，最终调用netif_receive_skb(skb)。1.7 netif_receive_skb函数

netif_receive_skb是链路层接收数据报的最后一站。它根据注册在全局数组ptype_all和ptype_base里的网络层数据报类型，把数据报递交给不同的网络层协议的接收函数(INET域中主要是ip_rcv和arp_rcv)。

Code

该函数主要就是调用第三层协议的接收函数处理该skb包，进入第三层网络层处理。