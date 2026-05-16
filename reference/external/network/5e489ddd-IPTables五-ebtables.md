---
title: IPTables五----ebtables
source: https://www.jianshu.com/p/79bcf09aed25
kind: external
domain: network
original_date: 2017-03-19
fetched_at: 2026-05-16
bookmark_title: IPTables五----ebtables - 简书
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.jianshu.com](https://www.jianshu.com/p/79bcf09aed25)
> 原始日期：2017-03-19
> 抓取日期：2026-05-16

# IPTables五----ebtables

五、ebtables

iptables可以对ip层报文进行操作，那么以太网层面呢？linux针对以太网网桥引入了ebtables，它在功能上和iptables相似（主要就是对以太网包进行操作）。它的出现为网桥设备设置防火墙带来了便利。在后文我们可以看到etables一般不会单独使用而在内核开启bridge-nf功能后和iptables一起工作。

首先在进入bridge之前会进行判定，此报文是不是需要进行bridge，当这个包不需要做bridge时刻，就直接走到Ｉｐ层的路由处理了。

只有以太网帧需要做bridging是才会进入到网桥内部，此时ebtable才会生效。

1、主要功能（表）：

过滤（filter），mac Nat和brouting

2、brouting

brouting是网桥的一种特殊工作模式，它可以根据配置对满足某些规则的包送入三层进行路由；也可以根据配置对满足某些规则的二层包进行bridge。（它指对于某些满足以太网包直接送入ip层（由ip层进行路由），而对于其他包继续送入到网桥内部进行以太网报文的处理。当brouting决定送给ip 层进行routing时，routing使用的ip地址是网桥下属的物理端口的ip地址。）

默认的brouting动作就是让数据包进入到bridge。

brouting是ebtable处理包的第一步。

2.1 brouting表支持的链

brout表只支持brouting链，brouting链的动作有：accept，drop，redirect，return。

注意：accept表示数据包送入bridge，drop表示数据包进入brouting的route。

2.2brouting 的brouting链的redirect动作和 prerouting的prerouting链redirect动作

brouting 的redirect是将目的mac地址设置为数据包接入接口的物理mac地址；nat表的prerouting链中redirect是将数据包目标mac地址设置为虚拟网桥的mac地址。

3、filter

ebtable的filter由三个链：input（帧发送给网桥自己的），output（网桥自己发出的或者route的），forward（网桥内部转发的）

4、nat

ebtable的nat由三个链：prerouting output(网桥自己发出或route的包) postrouting

下图总结了ebtable的所有链。

５、bridge-nf

ebtable最重要的应用是当内核开启了bridge-nf功能后，将iptables和ebtables都整合到二层处理里。后文将详细描述。