---
title: linux网络工具iproute2的使用简介
source: https://blog.csdn.net/astrotycoon/article/details/52317288
kind: external
domain: network
author: 成就一亿技术人
original_date: 2026-04-11
fetched_at: 2026-05-16
bookmark_title: linux网络工具iproute2的使用简介 - CSDN博客
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/astrotycoon/article/details/52317288)
> 作者：成就一亿技术人
> 原始日期：2026-04-11
> 抓取日期：2026-05-16

# linux网络工具iproute2的使用简介

**一、写本文的目的**

本文完全是自己在学习iproute2的过程中搜集的大杂烩，记录在这里，方便以后自己查询学习，图片都是来自网络，在此表示感谢！



**二、简单了解iproute2工具套装**

iproute2是linux下管理控制TCP/IP网络和流量控制的新一代工具包，旨在替代老派的工具链net-tools，即大家比较熟悉的ifconfig，arp，route，netstat等命令。


要说这两套工具本质的区别，应该是net-tools是通过procfs(/proc)和ioctl系统调用去访问和改变内核网络配置，而iproute2则通过netlink套接字接口与内核通讯。

其次，net-tools的用法给人的感觉是比较乱，而iproute2的用户接口相对net-tools来说相对来说，更加直观。比如，各种网络资源（如link、IP地址、路由和隧道等）均使用合适的对象抽象去定义，使得用户可使用一致的语法去管理不同的对象。

所以，net-tools和iproute2都需要去学习掌握了。

iproute2的核心命令是ip:





**三、iproute2的典型应用**

本小节，我会使用net-tools和iproute2的命令做对比，做到简单明了，分别演示如何去获取、配置和操作系统网络信息。

以下是net-tools和iproute2的大致对比：



**（一）网络接口相关**

（1） 查询所有已连接的网络接口（network interface）

**使用net-tools**:

```
root@astrol:~# ifconfig -a
eth0 Link encap:Ethernet HWaddr 00:0c:29:0d:ce:93
inet addr:192.168.6.138 Bcast:192.168.6.255 Mask:255.255.255.0
inet6 addr: fe80::20c:29ff:fe0d:ce93/64 Scope:Link
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:202741 errors:1 dropped:3312 overruns:0 frame:0
TX packets:60730 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:27472662 (27.4 MB) TX bytes:51025509 (51.0 MB)
Interrupt:18 Base address:0x2000
eth0:1 Link encap:Ethernet HWaddr 00:0c:29:0d:ce:93
inet addr:192.168.6.139 Bcast:192.168.6.255 Mask:255.255.255.0
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
Interrupt:18 Base address:0x2000
lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
inet6 addr: ::1/128 Scope:Host
UP LOOPBACK RUNNING MTU:65536 Metric:1
RX packets:5 errors:0 dropped:0 overruns:0 frame:0
TX packets:5 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:512 (512.0 B) TX bytes:512 (512.0 B)
```


ifconfig -a显示的是系统所有的网络接口，不管是激活的还是未激活的。

这里简单对ifconfig的输出做个解释：

第一行：Link encap（连接类型） HWaddr（网卡的硬件地址，即MAC地址）

第二行：inet addr（网卡的IPv4地址） Bcast（广播地址） Mask（子网掩码）

第三行：inet6 addr（网卡的IPv6地址）

第四行：UP（代表网卡是激活状态） BROADCAST（支持广播） RUNNING（代表网卡的网线被接上） MULTICAST（支持组播） MTU（最大传输单元） Metric（用于计算路由的成本）

第五、六行： 表示网络启动到现在接收和发送的网络包（packets）数量

第七行：collisions（冲突信息包的数目） txqueuelen（发送队列的大小）

第八行：表示网络启动到现在接收和发送的总字节量（bytes）

HWaddr :网卡的硬件地址，即MAC地址

inet addr：IPv4的IP 地址

Bcast：广播地址

mask：子网掩码

inet6 addr：IPv6地址

MTU:最大传输单元

Metric：用于计算路由的成本

RX：表示网络启动到现在的封包接受情况 (Receive)

packets:表示接包数

errors:表示接包发生错误的数量

dropped：表示丢弃的包数量

overruns:表示接收时因过速而丢失的数据包数

frame：表示发生frame错误而丢失的数据包数

TX：从网络启动到现在传送的情况 (Transmit)

collisions：冲突信息包的数目

txqueuelen：发送队列的大小

RX byte、TX byte:总传送/接受的量


注：由RX和TX可以了解网络是否非常繁忙

注：errors:0 dropped:0 overruns:0 frame:0，都为0 说明网络比较稳定

注：collisions发生太多次表示网络状况不太好


如果只想知道特定网络接口的信息，可以指定具体网络接口名称，例如ifconfig eth0，ifconfig lo



**使用iproute2**:

```
root@astrol:~# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
```


同样的，想查看特定网络接口的信息，直接指定网络接口名称即可。

```
root@astrol:~# ip link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
```


如果想让输出的结果像ifconfig那样详细，可以增加-s选项：
```
root@astrol:~# ip -s link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
RX: bytes packets errors dropped overrun mcast
40288764 244422 1 3651 0 0
TX: bytes packets errors dropped carrier collsns
51239397 62116 0 0 0 0
```


这样，就可以看到网络接口的流量信息了。

如果只想看当前被激活的网络接口，可以在命令后头增加一个up:

```
root@astrol:~# ip link show up
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
```


（2）查询网络设备的IP地址

使用net-tools：

`root@astrol:~# ifconfig eth0`


使用iproute2：

```
root@astrol:~# ip addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
inet 192.168.6.138/24 brd 192.168.6.255 scope global eth0
valid_lft forever preferred_lft forever
inet 192.168.6.139/24 brd 192.168.6.255 scope global secondary eth0:1
valid_lft forever preferred_lft forever
inet6 fe80::20c:29ff:fe0d:ce93/64 scope link
valid_lft forever preferred_lft forever
```


当不指定网络接口时，ip addr其实是ip addr show的简略写法。
（3）设置网络设备的IP地址

使用net-tools：

```
root@astrol:~# ifconfig eth0:1 192.168.6.140
root@astrol:~# ifconfig eth0:1 192.168.6.140 netmask 255.255.255.0
root@astrol:~# ifconfig eth0:1 192.168.6.140 netmask 255.255.255.0 broadcast 192.168.6.255
```


使用iproute2：

`root@astrol:~# ip addr add 192.168.6.140/24 brd + dev eth0:1`


这里使用的模版是：ip addr
add ip_address/net_prefix brd + dev
interface
net_prefix隐含指定了子网掩码，brd +表明是标准的广播地址。

需要了解的一点是，通过ip addr可以非常容易地给一块网卡添加多个地址，ifconfig同样可以，是通过叫做“IP别名”的方式做到的。

```
root@astrol:~# ip addr add 192.168.6.140/24 broadcast 192.168.6.255 dev eth0
root@astrol:~# ip addr add 192.168.6.141/24 broadcast 192.168.6.255 dev eth0
root@astrol:~# ip addr add 192.168.6.142/24 broadcast 192.168.6.255 dev eth0
```


（4）删除网络设备的IP地址

使用net-tools：

貌似没有什么好办法去做。

使用iproute2：

模版：ip addr del ip_address/net_prefix dev interface

```
root@astrol:~# ip -4 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 1000
inet 192.168.6.138/24 brd 192.168.6.255 scope global eth0
valid_lft forever preferred_lft forever
inet 192.168.6.141/24 brd 192.168.6.255 scope global secondary eth0
valid_lft forever preferred_lft forever
root@astrol:~# ip addr del 192.168.6.141/24 dev eth0
root@astrol:~# ip -4 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 1000
inet 192.168.6.138/24 brd 192.168.6.255 scope global eth0
valid_lft forever preferred_lft forever
```


此外，iproute2提供ip addr flush可以一次性删除一个网络设备的所有地址：
`root@astrol:~# ip addr flush dev eth0`


默认的，这条命令会删除IPv4和IPv6的地址，如果想分别删除，可以通过分别指定-4和-6选项。


（5）激活或者停用网络接口

使用net-tools：

```
root@astrol:~# ifconfig eth0 up
root@astrol:~# ifcofig eth0 dow
```


在linux下还可以使用ifup和ifdown来达到同样的目的。
使用iproute2：

```
root@astrol:~# ip link set eth0 up
root@astrol:~# ip link set eth0 down
```


（6）设置或者改变网络接口的参数（属性）

一个网络接口具体有哪些参数可以供我们去设置呢？输入ip link set eth0，然后按两次TAB键，如下：



可以看到其中的up和down就是用来激活或者停用某个网络接口的。例如，使能或者关闭eth0的多播功能：

使用net-tools：

```
root@astrol:~# ifconfig eth0 multicast
root@astrol:~# ifconfig eth0 -multicast
```


使用
iproute2：
```
root@astrol:~# ip link set eth0 multicast on
root@astrol:~# ip link set eth0 multicast off
```




通常，调整最大传输单元MTU用的比较多。

使用net-tools：

```
root@astrol:~# ifconfig eth0 mtu 1400
root@astrol:~# ip link show eth0
2: eth0: <BROADCAST,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
```


使用iproute2：

```
root@astrol:~# ip link set eth0 mtu 1500
root@astrol:~# ip link show eth0
2: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
link/ether 00:0c:29:0d:ce:93 brd ff:ff:ff:ff:ff:ff
```


改变网卡硬件地址，即MAC地址（注意，修改MAC地址前网卡必须先关闭）：

使用net-tools：

```
root@astrol:~# ifconfig eth0 down
root@astrol:~# ifconfig eth0 hw ether 00:0c:29:0d:ce:95 up
```


使用iproute2：

```
root@astrol:~# ip link set eth0 down
root@astrol:~# ip link set eth0 address 00:0c:29:0d:ce:95
root@astrol:~# ip link set eth0 up
```


类似的，需要先关闭网卡再设置的属性有name。
参考链接：

《Linux Advanced Routing & Traffic Control HOWTO》

《IPROUTE2 Utility Suite Howto》

《How To Use IPRoute2 Tools to Manage Network Configuration on a Linux VPS》

《Linux TCP/IP 网络工具对比：net-tools 和 iproute2》（英文原文：Linux TCP/IP networking: net-tools vs. iproute2）

《 iproute2学习笔记》