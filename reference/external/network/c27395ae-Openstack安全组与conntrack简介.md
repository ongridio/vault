---
title: Openstack安全组与conntrack简介
source: http://blog.csdn.net/bc_vnetwork/article/details/51350787
kind: external
domain: network
author: 成就一亿技术人
original_date: 2026-03-01
fetched_at: 2026-05-16
bookmark_title: Openstack安全组与conntrack简介 - CSDN博客
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](http://blog.csdn.net/bc_vnetwork/article/details/51350787)
> 作者：成就一亿技术人
> 原始日期：2026-03-01
> 抓取日期：2026-05-16

# Openstack安全组与conntrack简介

Openstack中的安全组实现相互信任的虚拟机之间的通信，绑定同一个安全组的虚拟机使用相同的安全策略。安全组作用范围是在虚拟机上，更具体来说是作用在虚拟机的端口而不是网络上。Openstack中安全组基于Iptables实现，由于当前OpenvSwitch(ovs)不能使用iptables rule，所以虚拟机先连接linux bridge，再连接到ovs网桥。参考链接[1]。



使用Iptables时，报文的状态可以归类为四种：NEW ESTABLISEDRELATED INVAILD. Netfilter就是通过conntrack实现连接追踪的。Conntrack是linux的一个内核模块，使用 conntrack entry记录连接的状态信息，具体可参考[2]。



**1. sg without conntrack-tool**



图1



Neutron最初实现安全组功能时并没有考虑对conntrack的处理，从而导致一些问题。如图1虚拟机vm1(1.1.1.8) vm2(1.1.1.9)属于同一子网，位于同一个计算节点，都绑定default安全组，执行vm1 ping vm2后能够ping通。此时删除default安全组中ingress规则保留egress规则，发现vm1 ping vm2仍未中断[3]。



究其原因，ICMP报文到达vm2连接的linux bridge时进入Iptables处理阶段。iptables中vm2对应的i链(参考[4])有一条规则为-mstate --state RELATED,ESTABLISHED -m comment -j RETURN(默认规则) ，表示状态为RELATED E