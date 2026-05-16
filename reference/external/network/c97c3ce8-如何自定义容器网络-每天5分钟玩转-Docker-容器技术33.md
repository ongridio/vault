---
title: 如何自定义容器网络？- 每天5分钟玩转 Docker 容器技术（33）
source: http://www.cnblogs.com/CloudMan6/p/7077198.html
kind: external
domain: network
author: CloudMan
original_date: 2017-06-26
fetched_at: 2026-05-16
bookmark_title: 如何自定义容器网络？- 每天5分钟玩转 Docker 容器技术（33） - CloudMan - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/CloudMan6/p/7077198.html)
> 作者：CloudMan
> 原始日期：2017-06-26
> 抓取日期：2026-05-16

# 如何自定义容器网络？- 每天5分钟玩转 Docker 容器技术（33）

除了 none, host, bridge 这三个自动创建的网络，用户也可以根据业务需要创建 user-defined 网络。


Docker 提供三种 user-defined 网络驱动：bridge, overlay 和 macvlan。overlay 和 macvlan 用于创建跨主机的网络，我们后面有章节单独讨论。

我们可通过 bridge 驱动创建类似前面默认的 bridge 网络，例如：


查看一下当前 host 的网络结构变化：


新增了一个网桥 `br-eaed97dc9a77`

，这里 `eaed97dc9a77`

正好新建 bridge 网络 `my_net`

的短 id。执行 `docker network inspect`

查看一下 `my_net`

的配置信息：


这里 172.18.0.0/16 是 Docker 自动分配的 IP 网段。

我们可以自己指定 IP 网段吗？

答案是：可以。

只需在创建网段时指定 `--subnet`

和 `--gateway`

参数：


这里我们创建了新的 bridge 网络 `my_net2`

，网段为 172.22.16.0/24，网关为 172.22.16.1。与前面一样，网关在 `my_net2`

对应的网桥 `br-5d863e9f78b6`

上：


容器要使用新的网络，需要在启动时通过 `--network`

指定：


容器分配到的 IP 为 172.22.16.2。

到目前为止，容器的 IP 都是 docker 自动从 subnet 中分配，我们能否指定一个静态 IP 呢？

答案是：可以，通过`--ip`

指定。


注：**只有使用 --subnet 创建的网络才能指定静态 IP**。

`my_net`

创建时没有指定 `--subnet`

，如果指定静态 IP 报错如下：


好了，我们来看看当前 docker host 的网络拓扑结构。


下一节讨论这几个容器之间的连通性。