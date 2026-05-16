---
title: TCP/IP协议中backlog参数
source: https://www.cnblogs.com/Orgliny/p/5780796.html
kind: external
domain: network
author: Orgliny
original_date: 2016-08-17
fetched_at: 2026-05-16
bookmark_title: TCP/IP协议中backlog参数 - Orgliny - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/Orgliny/p/5780796.html)
> 作者：Orgliny
> 原始日期：2016-08-17
> 抓取日期：2026-05-16

# TCP/IP协议中backlog参数

# TCP/IP协议中backlog参数

TCP建立连接是要进行三次握手，但是否完成三次握手后，服务器就处理（accept）呢？

backlog其实是一个连接队列，在Linux内核2.2之前，backlog大小包括半连接状态和全连接状态两种队列大小。

半连接状态为：服务器处于Listen状态时收到客户端SYN报文时放入半连接队列中，即SYN queue（服务器端口状态为：SYN_RCVD）。

全连接状态为：TCP的连接状态从服务器（SYN+ACK）响应客户端后，到客户端的ACK报文到达服务器之前，则一直保留在半连接状态中；当服务器接收到客户端的ACK报文后，该条目将从半连接队列搬到全连接队列尾部，即 accept queue （服务器端口状态为：ESTABLISHED）。

在Linux内核2.2之后，分离为两个backlog来分别限制半连接（SYN_RCVD状态）队列大小和全连接（ESTABLISHED状态）队列大小。

SYN queue 队列长度由 /proc/sys/net/ipv4/tcp_max_syn_backlog 指定，默认为2048。

Accept queue 队列长度由 /proc/sys/net/core/somaxconn 和使用listen函数时传入的参数，二者取最小值。默认为128。在Linux内核2.4.25之前，是写死在代码常量 SOMAXCONN ，在Linux内核2.4.25之后，在配置文件 /proc/sys/net/core/somaxconn 中直接修改，或者在 /etc/sysctl.conf 中配置 net.core.somaxconn = 128 。



可以通过ss命令来显示

[root@localhost ~]# ss -l State Recv-Q Send-Q Local Address:Port Peer Address:Port LISTEN 0 128 *:http *:* LISTEN 0 128 :::ssh :::* LISTEN 0 128 *:ssh *:* LISTEN 0 100 ::1:smtp :::* LISTEN 0 100 127.0.0.1:smtp *:*


在LISTEN状态，其中 Send-Q 即为Accept queue的最大值，Recv-Q 则表示Accept queue中等待被服务器accept()。

另外客户端connect()返回不代表TCP连接建立成功，有可能此时accept queue 已满，系统会直接丢弃后续ACK请求；客户端误以为连接已建立，开始调用等待至超时；服务器则等待ACK超时，会重传SYN+ACK 给客户端，重传次数受限 net.ipv4.tcp_synack_retries ，默认为5，表示重发5次，每次等待30~40秒，即半连接默认时间大约为180秒，该参数可以在tcp被洪水攻击是临时启用这个参数。

查看SYN queue 溢出

[root@localhost ~]# netstat -s | grep LISTEN 102324 SYNs to LISTEN sockets dropped

查看Accept queue 溢出

[root@localhost ~]# netstat -s | grep TCPBacklogDrop TCPBacklogDrop: 2334


参考资料：


作者：Orgliny

出处：https://www.cnblogs.com/Orgliny

本文采用**知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议**进行许可，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接。