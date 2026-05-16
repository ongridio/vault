---
title: 无状态TCP的ip_conntrack
source: https://www.cnblogs.com/dyllove98/archive/2013/07/13/3188425.html
kind: external
domain: network
author: Jlins
original_date: 2013-07-13
fetched_at: 2026-05-16
bookmark_title: 无状态TCP的ip_conntrack - jlins - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/dyllove98/archive/2013/07/13/3188425.html)
> 作者：Jlins
> 原始日期：2013-07-13
> 抓取日期：2026-05-16

# 无状态TCP的ip_conntrack

# 无状态TCP的ip_conntrack

Linux的ip_conntrack实现得过于沉重和精细。而实际上有时候，根本不需要在conntrack中对TCP的状态进行跟踪，只把它当UDP好了，我们的需求就是让系统可以将一个数据包和一个五元组标示的流相关联，因为很多的基于流的策略都设置在conntrack结构中，所以当关联好之后，就可以直接取出策略来作用于数据包了，不再需要为每一个数据包都来一次策略匹配。

TCP的状态本就不应该在途中被检测，这也是TCP/IP的设计者的中心思想。还好，Linux提供了两个sysctl参数可以在一定程度上放弃对TCP状态的精细化监控，它们是：

**net.netfilter.nf_conntrack_tcp_loose = 1net.netfilter.ip_conntrack_tcp_be_liberal = 1**相关的代码解释是：

/* "Be conservative in what you do, be liberal in what you accept from others." If it's non-zero, we mark only out of window RST segments as INVALID. */ static int nf_ct_tcp_be_liberal __read_mostly = 0; /* If it is set to zero, we disable picking up already established connections. */ static int nf_ct_tcp_loose __read_mostly = 1;


在ip_conntrack的逻辑中，如果一个包没有在conntrack哈希中找到对应的流，就会被认为是一个流的头包，流程如下：


TCP对数据包的要求太高了，在上述的loose参数不为1的情况下，tcp_new要求新到来的第一个包必然要有SYN标志，否则不予创建conntrack，数据包将成为游离数据包，NAT等都将失去作用，即便是中间的数据包，be_liberal不为1时，tcp_packet要求数据包必须in_window。这就是为何TCP的establish conntrack超时时间设置5天那么久的原因，还好，有上述两个参数，我们可以避开这个，将TCP尽可能地和UDP同等对待。我的方式更猛，索性将TCP的nf_conntrack_l4proto换成了nf_conntrack_l4proto_udp4，这样连sysctl参数都不用设置了。

**NAT问题的解释**

一条TCP连接，经过Linux Box，被NAT，过了120秒+之后，连接断开！这个解释很简单，我将net.netfilter.nf_conntrack_tcp_timeout_established设置成了120秒，此后conntrack超时被删除！再有数据时，按照上面的tcp_new流程，数据包将成为游离数据包，NAT依赖conntrack，故失效，连接依赖NAT，故而连接失效！