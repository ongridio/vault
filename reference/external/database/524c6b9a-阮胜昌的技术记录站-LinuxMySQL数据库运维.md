---
title: 阮胜昌的技术记录站-LinuxMySQL数据库运维
source: http://www.linuxmysql.com/16/2018/959.htm
kind: external
domain: database
original_date: 2018-11-01
fetched_at: 2026-05-16
bookmark_title: centos 6.9用openswan ipsec和xl2tpd搭建l2tp VPN
tags: [external, database]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.linuxmysql.com](http://www.linuxmysql.com/16/2018/959.htm)
> 原始日期：2018-11-01
> 抓取日期：2026-05-16

# 阮胜昌的技术记录站-LinuxMySQL数据库运维

一键安装包下载地址：http://www.linuxmysql.com/tools/l2tp.sh

第一步：准备工作

首先的首先，给系统的软件都升一下级:(可选)

yum update

再来，安装下面需要的软件，和编辑和编译环境相关的，都装上吧，下面有些在安装时会用到，比如lsof之类。

yum install -y make gcc gmp-devel xmlto bison flex xmlto libpcap-devel lsof vim-enhanced man

第二步：安装

下面正式开始安装l2tp VPN!

yum install openswan ppp xl2tpd

如果找不到的软件，那么可以去http://pkgs.org/找rpm包。比如xl2tpd我就找不到，去pkgs.org就可以搜到：

++++++++++++++++++++++++++++++++++++++

CentOS 6

Atomic:

xl2tpd-1.2.7-1.el6.art.i686.rpm

Layer 2 Tunnelling Protocol Daemon (RFC 2661)

xl2tpd-1.2.7-1.el6.art.x86_64.rpm

Layer 2 Tunnelling Protocol Daemon (RFC 2661)

EPEL:

xl2tpd-1.3.1-7.el6.i686.rpm

Layer 2 Tunnelling Protocol Daemon (RFC 2661)

xl2tpd-1.3.1-7.el6.x86_64.rpm

Layer 2 Tunnelling Protocol Daemon (RFC 2661)

根据自己的系统是i686还是x86_64找个对应的最新的xl2tpd-1.3.1-7下下来安装。其他包找不到同理一样做。都要找最新的！

yum install xl2tpd-1.3.1-7.el6.x86_64.rpm

++++++++++++++++++++++++++++++++++++++

第三步：配置

1. 编辑 /etc/ipsec.conf

#>/etc/ipsec.conf

#vim /etc/ipsec.conf

把下面xx.xxx.xxx.xxx换成你自己VPS实际的外网固定IP。其他的不动。

config setup

nat_traversal=yes

virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12

oe=off

protostack=netkey


conn L2TP-PSK-NAT

rightsubnet=vhost:%priv

also=L2TP-PSK-noNAT


conn L2TP-PSK-noNAT

authby=secret

pfs=no

auto=add

keyingtries=3

rekey=no

ikelifetime=8h

keylife=1h

type=transport

left=xxx.xxx.xxx.xxx

leftprotoport=17/1701

right=%any

rightprotoport=17/%any

2. 编辑/etc/ipsec.secrets

vim /etc/ipsec.secrets

xxx.xxx.xxx.xxx %any: PSK "YourPsk"

xx.xxx.xxx.xxx换成你自己VPS实际的外网固定IP, YourPsk你自己定一个，到时候连VPN的时候可以用，比如可以填vpn

注意空格。

3. 修改/添加 /etc/sysctl.conf

vim /etc/sysctl.conf

确保下面的字段都有，对应的值或下面一样。省事的话直接在/etc/sysctl.conf的末尾直接把下面内容的粘过去。

net.ipv4.ip_forward = 1

net.ipv4.conf.default.rp_filter = 0

net.ipv4.conf.all.send_redirects = 0

net.ipv4.conf.default.send_redirects = 0

net.ipv4.conf.all.log_martians = 0

net.ipv4.conf.default.log_martians = 0

net.ipv4.conf.default.accept_source_route = 0

net.ipv4.conf.all.accept_redirects = 0

net.ipv4.conf.default.accept_redirects = 0

net.ipv4.icmp_ignore_bogus_error_responses = 1

4.让修改后的sysctl.conf生效

sysctl -p

可能会有一些ipv6的错误，不用管他

error: "net.bridge.bridge-nf-call-ip6tables" is an unknown key

error: "net.bridge.bridge-nf-call-iptables" is an unknown key

error: "net.bridge.bridge-nf-call-arptables" is an unknown key

5. 验证ipsec运行状态

ipsec setup restart

ipsec verify

verify的内容如下所示，那么就离成功不远了。

Checking your system to see if IPsec got installed and started correctly:

Version check and ipsec on-path [OK]

Linux Openswan U2.6.32/K2.6.32-431.11.2.el6.i686 (netkey)

Checking for IPsec support in kernel [OK]

SAref kernel support [N/A]

NETKEY: Testing for disabled ICMP send_redirects [OK]

NETKEY detected, testing for disabled ICMP accept_redirects [OK]

Checking that pluto is running [OK]

Pluto listening for IKE on udp 500 [OK]

Pluto listening for NAT-T on udp 4500 [OK]

Checking for 'ip' command [OK]

Checking /bin/sh is not /bin/dash [OK]

Checking for 'iptables' command [OK]

Opportunistic Encryption Support [DISABLED]

如果有问题，你就得上网搜解决办法，否则下面进行不下去。比如有时你得把selinux关掉

把SELINUX设置成disabled，然后reboot一下机器就生效了。

vim /etc/sysconfig/selinux

SELINUX=disabled

6. 编辑 /etc/xl2tpd/xl2tpd.conf

vim /etc/xl2tpd/xl2tpd.conf

[global]

ipsec saref = yes

listen-addr = xxx.xxx.xxx.xxx ;服务器地址

[lns default]

ip range = 10.1.1.2-10.1.1.100 ;这里是VPN client的内网ip地址范围

local ip = 10.1.1.1 ;这里是VPN server的内网地址

refuse chap = yes

refuse pap = yes

require authentication = yes

ppp debug = yes

pppoptfile = /etc/ppp/options.xl2tpd

length bit = yes

7. 编辑 /etc/ppp/options.xl2tpd

vim /etc/ppp/options.xl2tpd

require-mschap-v2

ms-dns 8.8.8.8

ms-dns 8.8.4.4

asyncmap 0

auth

crtscts

lock

hide-password

modem

debug

name l2tpd

proxyarp

lcp-echo-interval 30

lcp-echo-failure 4

8。 配置用户名,密码:编辑 /etc/ppp/chap-secrets

vim /etc/ppp/chap-secrets

client和secret自己填，server和IP留*号，

# Secrets for authentication using CHAP

# client server secret IP addresses

username * userpass *

9. 重启xl2tp

service xl2tpd restart

10. 开放端口以及转发

原样执行下面所有命令，

#Allow ipsec traffic

iptables -A INPUT -m policy --dir in --pol ipsec -j ACCEPT

iptables -A FORWARD -m policy --dir in --pol ipsec -j ACCEPT

#Do not NAT VPN traffic

iptables -t nat -A POSTROUTING -m policy --dir out --pol none -j MASQUERADE

#Forwarding rules for VPN

iptables -A FORWARD -i ppp+ -p all -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

#Ports for Openswan / xl2tpd

iptables -A INPUT -m policy --dir in --pol ipsec -p udp --dport 1701 -j ACCEPT

iptables -A INPUT -p udp --dport 500 -j ACCEPT

iptables -A INPUT -p udp --dport 4500 -j ACCEPT

iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

再执行下面保存iptables

service iptables save

service iptables restart

11. 添加自启动

chkconfig xl2tpd on

chkconfig iptables on

chkconfig ipsec on

12. 重启

reboot


调试：

连不上的时候先关闭iptables来调试。

service iptables stop

确定能连上以后再打开iptables

service iptables start

如果这时连不上了，那么就是iptables的问题了

特别注意iptables里的顺序， INPUT和FORWARD里的REJECT一定是写在最后面，否则写在他们之后的port就都被REJECT了！

下面是我自己的iptables，可供参考

*nat

:PREROUTING ACCEPT [82:15507]

:POSTROUTING ACCEPT [0:0]

:OUTPUT ACCEPT [6:447]

-A POSTROUTING -m policy --dir out --pol none -j MASQUERADE

-A POSTROUTING -s 10.1.1.0/24 -o eth0 -j MASQUERADE

COMMIT

# Completed on Fri Apr 4 05:44:30 2014

# Generated by iptables-save v1.4.7 on Fri Apr 4 05:44:30 2014

*filter

:INPUT ACCEPT [0:0]

:FORWARD ACCEPT [0:0]

:OUTPUT ACCEPT [490:286471]

-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

-A INPUT -p icmp -j ACCEPT

-A INPUT -i lo -j ACCEPT

-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT

-A INPUT -m policy --dir in --pol ipsec -j ACCEPT

-A INPUT -p udp -m policy --dir in --pol ipsec -m udp --dport 1701 -j ACCEPT

-A INPUT -p udp -m udp --dport 500 -j ACCEPT

-A INPUT -p udp -m udp --dport 4500 -j ACCEPT

-A INPUT -p esp -j ACCEPT

-A INPUT -j REJECT --reject-with icmp-host-prohibited

-A FORWARD -m policy --dir in --pol ipsec -j ACCEPT

-A FORWARD -i ppp+ -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT

-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

-A FORWARD -j REJECT --reject-with icmp-host-prohibited

COMMIT

客户端连接：通过客户端

在win7上测试连接***

打开网络共享中心--设置新的连接或网络--连接到工作区--创建***

点击右下角网络图标--***连接--点击属性

1.常规中的ip为vps的公网ip

2.***类型：使用Ipsec 的第二层隧道协议(L2TP/IPsec)

数据加密：需要加(如果服务器拒绝将断开连接)

高级设置：使用预共享的密钥作身份验证

密钥：YourPsk (前面设置过的)

点击确定后输入用户名和密码即可登陆。