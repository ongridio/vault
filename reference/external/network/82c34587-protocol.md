---
title: protocol
source: https://zhangbinalan.gitbooks.io/protocol/content/tcpan_quan_wen_ti.html
kind: external
domain: network
original_date: 2016-04-24
fetched_at: 2026-05-16
bookmark_title: TCP安全问题 | protocol
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[zhangbinalan.gitbooks.io](https://zhangbinalan.gitbooks.io/protocol/content/tcpan_quan_wen_ti.html)
> 原始日期：2016-04-24
> 抓取日期：2026-05-16

# protocol

## TCP安全问题

### SYN攻击

#### 什么是SYN攻击（SYN Flood）

在三次握手过程中，服务器发送 SYN-ACK 之后，收到客户端的 ACK 之前的 TCP 连接称为半连接(half-open connect)此时服务器处于 SYN_RCVD 状态。当收到 ACK 后，服务器才能转入 ESTABLISHED 状态.

SYN 攻击指的是，攻击客户端在短时间内伪造大量不存在的IP地址，向服务器不断地发送SYN包，服务器回复确认包，并等待客户的确认。由于源地址是不存在的，服务器需要不断的重发直至超时，这些伪造的SYN包将长时间占用未连接队列，正常的SYN请求被丢弃，导致目标系统运行缓慢，严重者会引起网络堵塞甚至系统瘫痪

SYN 攻击是一种典型的 DoS/DDoS 攻击。

#### 如何检测 SYN 攻击？

检测 SYN 攻击非常的方便，当你在服务器上看到大量的半连接状态时，特别是源IP地址是随机的，基本上可以断定这是一次SYN攻击。在Linux/Unix上可以使用系统自带的netstats命令来检测SYN攻击

#### 如何防御SYN攻击

SYN攻击不能完全被阻止，除非将TCP协议重新设计。我们所做的是尽可能的减轻SYN攻击的危害，常见的防御 SYN 攻击的方法有如下几种：

- 缩短超时（SYN Timeout）时间
- 增加最大半连接数
- 过滤网关防护
- SYN cookies技术