---
title: Tracert not taking the right gateway
source: https://community.spiceworks.com/topic/569683-tracert-not-taking-the-right-gateway
kind: external
domain: network
author: Charles-Andremorin
original_date: 2014-08-26
fetched_at: 2026-05-16
bookmark_title: [SOLVED] Tracert not taking the right gateway - Windows Forum - Spiceworks
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[community.spiceworks.com](https://community.spiceworks.com/topic/569683-tracert-not-taking-the-right-gateway)
> 作者：Charles-Andremorin
> 原始日期：2014-08-26
> 抓取日期：2026-05-16

# Tracert not taking the right gateway

Well for starters, what network are you on? The first path is going to be out of your network then to the next network. I assume you on the 10.0.8.x network so that will be the first hop before it moves on to the next route.

You are defining a gateway that is outside your current subnet. With an IP of 10.0.8.14 and a mask of 255.255.252.0 (ie: /22) your computer can only directly talk to IPs in the range of 10.0.8.1 - 10.0.11.254. (10.0.8.0 being the network address, and 10.0.11.255 being the broadcast address.)

To use a gateway of 10.0.3.254 for anything your local IP would have to be in the range of 10.0.0.1 - 10.0.3.253, or your subnet mask would have to be 255.255.240.0 (ie: /20)

It would seem to me, that you would have to add a second NIC that connects to that default Gateway directly. IF you have a single NIC, the physical path takes it to the 10.0.8.254 Router first.