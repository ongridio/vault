---
title: 结合netstat和awk命令来统计网络连接数
source: http://www.cnblogs.com/derekchen/archive/2011/02/26/1965839.html
kind: external
domain: network
original_date: 2011-02-26
fetched_at: 2026-05-16
bookmark_title: 结合netstat和awk命令来统计网络连接数 - 晓风残梦 - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/derekchen/archive/2011/02/26/1965839.html)
> 原始日期：2011-02-26
> 抓取日期：2026-05-16

# 结合netstat和awk命令来统计网络连接数

# 结合netstat和awk命令来统计网络连接数

From: http://hi.baidu.com/thinkinginlamp/blog/item/afbcab64b1ad81f3f6365453.html

`netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'`


会得到类似下面的结果，具体数字会有所不同：

LAST_ACK 1


SYN_RECV 14

ESTABLISHED 79

FIN_WAIT1 28

FIN_WAIT2 3

CLOSING 5

TIME_WAIT 1669

状态：描述

CLOSED：无连接是活动的或正在进行

LISTEN：服务器在等待进入呼叫

SYN_RECV：一个连接请求已经到达，等待确认

SYN_SENT：应用已经开始，打开一个连接

ESTABLISHED：正常数据传输状态

FIN_WAIT1：应用说它已经完成

FIN_WAIT2：另一边已同意释放

ITMED_WAIT：等待所有分组死掉

CLOSING：两边同时尝试关闭

TIME_WAIT：另一边已初始化一个释放

LAST_ACK：等待所有分组死掉

也就是说，这条命令可以把当前系统的网络连接状态分类汇总。

下面解释一下为啥要这样写：

一个简单的管道符连接了netstat和awk命令。

------------------------------------------------------------------

先来看看netstat：

**netstat -n**

Active Internet connections (w/o servers)

Proto Recv-Q Send-Q Local Address Foreign Address State

tcp 0 0 123.123.123.123:80 234.234.234.234:12345 TIME_WAIT

你实际执行这条命令的时候，可能会得到成千上万条类似上面的记录，不过我们就拿其中的一条就足够了。

------------------------------------------------------------------

再来看看awk：

**/^tcp/**

滤出tcp开头的记录，屏蔽udp, socket等无关记录。

**state[]**

相当于定义了一个名叫state的数组

**NF**

表示记录的字段数，如上所示的记录，NF等于6

**$NF**

表示某个字段的值，如上所示的记录，$NF也就是$6，表示第6个字段的值，也就是TIME_WAIT

**state[$NF]**

表示数组元素的值，如上所示的记录，就是state[TIME_WAIT]状态的连接数

**++state[$NF]**

表示把某个数加一，如上所示的记录，就是把state[TIME_WAIT]状态的连接数加一

**END**

表示在最后阶段要执行的命令

**for(key in state)**

遍历数组

**print key,"\t",state[key]**

打印数组的键和值，中间用\t制表符分割，美化一下。

如发现系统存在大量TIME_WAIT状态的连接，通过调整内核参数解决，

`vim /etc/sysctl.conf`


编辑文件，加入以下内容：

`1.`

`net.ipv4.tcp_syncookies = 1 `

`2.`

`net.ipv4.tcp_tw_reuse = 1 `

`3.`

`net.ipv4.tcp_tw_recycle = 1 `

`4.`

`net.ipv4.tcp_fin_timeout = 30`

然后执行 `/sbin/sysctl -p`

让参数生效。

**net.ipv4.tcp_syncookies = 1** 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

**net.ipv4.tcp_tw_reuse = 1** 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

**net.ipv4.tcp_tw_recycle = 1** 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。

**net.ipv4.tcp_fin_timeout** 修改系統默认的 TIMEOUT 时间

下面附上TIME_WAIT状态的意义：

客户端与服务器端建立TCP/IP连接后关闭SOCKET后，服务器端连接的端口

状态为TIME_WAIT

是不是所有执行主动关闭的socket都会进入TIME_WAIT状态呢？

有没有什么情况使主动关闭的socket直接进入CLOSED状态呢？

主动关闭的一方在发送最后一个 ack 后

就会进入 TIME_WAIT 状态 停留2MSL（max segment lifetime）时间

这个是TCP/IP必不可少的，也就是“解决”不了的。

也就是TCP/IP设计者本来是这么设计的

主要有两个原因

1。防止上一次连接中的包，迷路后重新出现，影响新连接

（经过2MSL，上一次连接中所有的重复包都会消失）

2。可靠的关闭TCP连接

在主动关闭方发送的最后一个 ack(fin) ，有可能丢失，这时被动方会重新发

fin, 如果这时主动方处于 CLOSED 状态 ，就会响应 rst 而不是 ack。所以

主动方要处于 TIME_WAIT 状态，而不能是 CLOSED 。

TIME_WAIT 并不会占用很大资源的，除非受到攻击。

还有，如果一方 send 或 recv 超时，就会直接进入 CLOSED 状态

`netstat -an | grep SYN | awk '{print $5}' | awk -F: '{print $1}' | sort | uniq -c | sort -nr | more`


-
netstat -tna | cut -b 49- |grep TIME_WAIT | sort


取出目前所有 TIME_WAIT 的连接 IP ( 排序过 ) - net.ipv4.tcp_syncookies = 1 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

net.ipv4.tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。

net.ipv4.tcp_fin_timeout = 30 表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。

net.ipv4.tcp_keepalive_time = 1200 表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为20分钟。

net.ipv4.ip_local_port_range = 1024 65000 表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。

net.ipv4.tcp_max_syn_backlog = 8192 表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。

net.ipv4.tcp_max_tw_buckets = 5000 表示系统同时保持TIME_WAIT套接字的最大数量，如果超过这个数字，TIME_WAIT套接字将立刻被清除并打印警告信息。默认为180000，改 为5000。对于Apache、Nginx等服务器，上几行的参数可以很好地减少TIME_WAIT套接字数量，但是对于Squid，效果却不大。此项参 数可以控制TIME_WAIT套接字的最大数量，避免Squid服务器被大量的TIME_WAIT套接字拖死。在怀疑有Dos攻击的时候，可以输入

`1`

`netstat`

`-na |`

`grep`

`:80 |`

`awk`

`'{print $5}'`

`|`

`awk`

`-F`

`'::ffff:'`

`'{print $2}'`

`|`

`grep`

`':'`

`|`

`awk`

`-F:`

`'{print $1}'`

`|`

`sort`

`|`

`uniq`

`-c |`

`sort`

`-r |`

`awk`

`-F`

`' '`

`'{if ($1 > 50) print $2}'`

`|`

`sed`

`'s/^.*$/iptables -I firewall 1 -p tcp -s & --dport 80 --syn -j REJECT/'`

`| sh`

先把冲击量最大的前50个IP给封了.

还可以加几个例外的白名单IP

以上的命令还没有实际考证过, 放在这里做参考.`1`

`netstat`

`-na |`

`grep`

`:80 |`

`awk`

`'{print $5}'`

`|`

`awk`

`-F`

`'::ffff:'`

`'{print $2}'`

`|`

`grep`

`':'`

`|`

`awk`

`-F:`

`'{print $1}'`

`|`

`sort`

`|`

`uniq`

`-c |`

`sort`

`-r |`

`awk`

`-F`

`' '`

`'{if ($1 > 50) print $2}'`

`|`

`grep`

`-`

`v`

`xxx.xxx.xxx.xxx |`

`sed`

`'s/^.*$/iptables -I RH-Firewall-1-INPUT 1 -p tcp -m tcp -s & --dport 80 --syn -j REJECT/'`

`| sh`