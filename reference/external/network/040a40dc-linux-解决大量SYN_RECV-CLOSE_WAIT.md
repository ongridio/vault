---
title: linux 解决大量SYN_RECV CLOSE_WAIT
source: https://blog.csdn.net/bigtree_3721/article/details/51278836
kind: external
domain: network
author: 成就一亿技术人
original_date: 2016-04-29
fetched_at: 2026-05-16
bookmark_title: linux 解决大量SYN_RECV CLOSE_WAIT - CSDN博客
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/bigtree_3721/article/details/51278836)
> 作者：成就一亿技术人
> 原始日期：2016-04-29
> 抓取日期：2026-05-16

# linux 解决大量SYN_RECV CLOSE_WAIT

###

[root@localhost ~]# netstat -nat | awk '/^tcp/{++S[$NF]}END{for (a in S) print a,S[a]}'

[root@localhost ~]# netstat -ant

Active Internet connections (servers and established)

Proto Recv-Q Send-Q Local Address Foreign Address State

tcp 0 0 0.0.0.0:199 0.0.0.0:*
LISTEN

tcp 0 0 0.0.0.0:3306 0.0.0.0:*
LISTEN

tcp 0 0 0.0.0.0:80 0.0.0.0:*
LISTEN

tcp 0 0 61.X.X.X:80 118.133.52.200:28521 SYN_RECV

tcp 0 0 61.X.X.X:80 123.133.15.30:1953 SYN_RECV

tcp 0 0 61.X.X.X:80 113.92.56.108:1734 SYN_RECV

tcp 0 0 61.X.X.X:80 222.133.148.238:2140 SYN_RECV

tcp 0 0 61.X.X.X:80 207.46.199.45:23806 SYN_RECV

tcp 0 0 61.X.X.X:80 124.115.0.167:50044 SYN_RECV

tcp 0 0 61.X.X.X:80 222.85.70.53:28060 SYN_RECV

tcp 0 0 61.X.X.X:80 207.46.199.50:48724 SYN_RECV

tcp 0 0 61.X.X.X:80 67.195.114.214:36812 SYN_RECV

tcp 0 0 61.X.X.X:80 67.195.114.214:33046 SYN_RECV

tcp 0 0 61.X.X.X:80 123.154.64.248:10707 SYN_RECV

tcp 0 0 61.X.X.X:80 121.235.34.148:2879 SYN_RECV

tcp 0 0 61.X.X.X:80 60.176.205.81:1059 SYN_RECV

tcp 0 0 61.X.X.X:80 119.115.114.140:1930 SYN_RECV

tcp 0 0 0.0.0.0:21 0.0.0.0:*
LISTEN

tcp 1 0 61.X.X.X:80 67.195.37.179:32800 CLOSE_WAIT

tcp 399 0 61.X.X.X:80 67.195.114.214:48898 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.114.214:41998 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.114.214:39951 CLOSE_WAIT

tcp 397 0 61.X.X.X:80 58.33.190.212:64535 ESTABLISHED

tcp 1 0 61.X.X.X:80 67.195.114.214:42790 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:58980 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 110.75.169.119:37211 CLOSE_WAIT

tcp 0 0 61.X.X.X:80 60.211.114.236:54143 ESTABLISHED

tcp 1 0 61.X.X.X:80 67.195.114.214:36443 CLOSE_WAIT

tcp 297 0 61.X.X.X:80 114.96.73.171:58300 ESTABLISHED

tcp 598 0 61.X.X.X:80 113.8.3.223:2796 ESTABLISHED

tcp 393 0 61.X.X.X:80 67.195.114.214:47940 CLOSE_WAIT

tcp 393 0 61.X.X.X:80 67.195.114.214:45125 CLOSE_WAIT

tcp 405 0 61.X.X.X:80 67.195.114.214:51274 CLOSE_WAIT

tcp 397 0 61.X.X.X:80 67.195.37.179:42364 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:55932 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 116.226.23.211:30002 CLOSE_WAIT

tcp 564 0 61.X.X.X:80 117.84.15.173:42721 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:35144 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:57164 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 220.181.7.117:43809 CLOSE_WAIT

tcp 395 0 61.X.X.X:80 67.195.37.179:38478 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:59297 CLOSE_WAIT

tcp 230 0 61.X.X.X:80 110.75.169.91:39862 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 122.230.79.109:8392 CLOSE_WAIT

tcp 399 0 61.X.X.X:80 67.195.114.214:51102 CLOSE_WAIT

tcp 405 0 61.X.X.X:80 67.195.114.214:46976 CLOSE_WAIT

tcp 399 0 61.X.X.X:80 67.195.114.214:46724 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:34999 CLOSE_WAIT

tcp 399 0 61.X.X.X:80 67.195.114.214:44723 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.114.214:37817 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.114.214:38590 CLOSE_WAIT

tcp 689 0 61.X.X.X:80 125.126.42.74:61242 CLOSE_WAIT

tcp 405 0 61.X.X.X:80 67.195.114.214:49085 CLOSE_WAIT

tcp 395 0 61.X.X.X:80 67.195.37.179:42899 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:32915 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 203.208.60.210:45055 CLOSE_WAIT

tcp 401 0 61.X.X.X:80 183.0.229.195:2538 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:54940 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 110.75.169.123:35024 CLOSE_WAIT

tcp 395 0 61.X.X.X:80 67.195.37.179:36323 CLOSE_WAIT

tcp 241 0 61.X.X.X:80 61.172.128.168:15674 ESTABLISHED

tcp 0 0 61.X.X.X:80 220.181.49.3:30687 ESTABLISHED

tcp 0 0 61.X.X.X:80 115.170.147.119:1815 ESTABLISHED

tcp 1 0 61.X.X.X:80 67.195.114.214:40688 CLOSE_WAIT

tcp 395 0 61.X.X.X:80 67.195.37.179:40643 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.114.214:44020 CLOSE_WAIT

tcp 393 0 61.X.X.X:80 67.195.114.214:50170 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:33226 CLOSE_WAIT

tcp 239 0 61.X.X.X:80 110.75.169.91:58591 CLOSE_WAIT

tcp 0 0 61.X.X.X:80 58.83.127.175:39686 ESTABLISHED

tcp 1 0 61.X.X.X:80 67.195.37.179:56784 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:58833 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.195.37.179:54995 CLOSE_WAIT

tcp 250 0 61.X.X.X:80 202.160.180.14:46439 CLOSE_WAIT

tcp 397 0 61.X.X.X:80 67.195.37.179:40155 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 118.116.15.201:1082 CLOSE_WAIT

tcp 582 0 61.X.X.X:80 119.1.54.103:2009 CLOSE_WAIT

tcp 0 0 61.X.X.X:80 116.22.222.167:52342 ESTABLISHED

tcp 1 0 61.X.X.X:80 110.75.169.178:36424 CLOSE_WAIT

tcp 251 0 61.X.X.X:80 110.75.169.148:39031 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 67.218.116.131:33831 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 110.75.169.124:54417 CLOSE_WAIT

tcp 238 0 61.X.X.X:80 124.115.0.169:39740 CLOSE_WAIT

tcp 1 0 61.X.X.X:80 110.75.169.150:60568 CLOSE_WAIT


防止同步包洪水（Sync Flood）

# iptables -A FORWARD -p tcp --syn -m limit --limit 1/s -j ACCEPT

也有人写作

#iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT

--limit 1/s 限制syn并发数每秒1次，可以根据自己的需要修改

防止各种端口扫描

# iptables -A FORWARD -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT

Ping洪水攻击（Ping of Death）

# iptables -A FORWARD -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT


vi /etc/sysctl.conf

net.ipv4.tcp_tw_reuse = 1

该文件表示是否允许重新应用处于TIME-WAIT状态的socket用于新的TCP连接。

net.ipv4.tcp_tw_recycle = 1

recyse是加速TIME-WAIT sockets回收

对tcp_tw_reuse和tcp_tw_recycle的修改，可能会出现.warning, got duplicate tcp line warning, got BOGUS tcp line.上面这二个参数指的是存在这两个完全一样的TCP连接，这会发生在一个连接被迅速的断开并且重新连接的情况，而且使用的端口和地址相同。但基本上这样的事情不会发生，无论如何，使能上述设置会增加重现机会。这个提示不会有人和危害，而且也不会降低系统性能，目前正在进行工作

net.ipv4.tcp_syncookies = 1

表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_synack_retries = 1

net.ipv4.tcp_keepalive_time = 1200

表示当keepalive起用的时候,TCP发送keepalive消息的频度。缺省是2小时

net.ipv4.tcp_fin_timeout = 30

fin_wait1状态是在发起端主动要求关闭tcp连接，并且主动发送fin以后，等待接收端回复ack时候的状态。对于本端断开的socket连接，TCP保持在FIN-WAIT-2状态的时间。对方可能会断开连接或一直不结束连接或不可预料的进程死亡。

net.ipv4.ip_local_port_range = 1024 65000

net.ipv4.tcp_max_syn_backlog = 8192

该文件指定了，在接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。

net.ipv4.tcp_max_tw_buckets = 5000

sysctl -p