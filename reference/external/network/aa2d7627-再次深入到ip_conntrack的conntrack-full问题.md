---
title: 再次深入到ip_conntrack的conntrack full问题
source: http://blog.csdn.net/dog250/article/details/7262619
kind: external
domain: network
author: 成就一亿技术人
original_date: 2026-05-15
fetched_at: 2026-05-16
bookmark_title: 再次深入到ip_conntrack的conntrack full问题 - Netfilter,iptables/OpenVPN/TCP guard:-( - CSDN博客
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](http://blog.csdn.net/dog250/article/details/7262619)
> 作者：成就一亿技术人
> 原始日期：2026-05-15
> 抓取日期：2026-05-16

# 再次深入到ip_conntrack的conntrack full问题

增加nf_conntrack_max固然可以缓解这个问题，或者说减小conntrack表项占据内核内存的时间也可以缓解之，然而这种补救措施都是治标不治本的.



然后在客户端上执行echo $sec /proc/sys/net/ipv4/netfilter/ip_conntrack_udp_timeout

其中sec要比服务器端的sleep参数更小即可。

如此udp客户端将收不到服务器eho回来的字符串，因为客户端只是放行状态为establish的入流量，如果ip_conntrack_udp_timeout配置过于短暂，NEW状态的conntrack过早被释放，这样将不会有establish状态的流量了。对于UDP而言，由于它是不确认无连接允许丢包的，因此影响还不是很大，TCP也有类似的问题，那就是如果你连接一个很远的且网络状况很恶劣的TCP服务器，然后你把ip_conntrack_tcp_timeout_synsent设置很小，这样就几乎完不成三次握手了，更进一步，如果你把ip_conntrack_tcp_timeout_established设置过小，那么一旦三次握手建立连接之后，客户端和服务器之间很久不发包，当establish状态到期后，conntrack被释放，此时服务器端主动发来一个包，该包的conntrack状态会是什么呢？因此给予tcp的establish状态5天的时间，是可以理解的。需要注意的是，对于tcp而言，由于无法简单的控制服务器发送syn-ack的延时，因此需要在establish状态而不是new状态做文章了(实际上，ip_conntrack的establish状态映射成了tcp的多个状态，包括syn-ack，ack，established)，试试看，效果和udp的一样。

前面关于ip_conntrack扯的太远了，我们的首要问题是conntrack full的问题。实际上，如果深入思考这个conntrack full的问题，就会发现，并不是conntrack容量太小或者表项保留时间过长引发的full。现实中的万事万物都不是无限的，对于计算机资源而言，更应该节约使用，不能让无关人士浪费这种资源，另外既然内核默认了一个表项的存活时间，那肯定是经过测试的经验值，自有它的道理。因此本质问题在于很多不需要conntrack的包也被conntrack了，这样就会挤掉很多真正需要conntrack的流量。

那么都是哪些流量需要conntrack呢？常用的就两个，一个是任何使用ctstate或者state这些match的iptables规则，另外一个就是所有的iptables的nat表中的规则，如果我们事先知道哪些流量需要使用iptables的[ct]state来控制，并且也知道哪些流量需要做NAT，那么余下的流量就都是和conntrack无关的流量了，可以不被ip_conntrack来跟踪。

幸运的是，Linux的Netfilter在PREROUTING以及OUTPUT这两个HOOK的conntrack之前安插了一个优先级更高的table，那就是raw，通过它就可以分离出不需要被conntrack的流量。如果你确定只有某个网卡进来的流量才需要做NAT，那么就执行下面的规则：




可见，必要时同时采取三种方式比较有效：1.增大conntrack_max;2.减少状态保存时间;3.分离无关流量。然而除了第三种方式，其余两种方式在操作时必须给自己十足的理由那么做才行，对于1，比必须明白内核内存被占有的方式，对于2，看看本文的前半部分。



#### 注解：不要过度减小NEW以及TCP的establish的CT状态的timeout的原因

尽量不要减小NEW状态时间，因为对于某些恶劣的网络，一个数据包的来回确实需要很长时间，对于TCP而言，此时RTT还没有测量呢。如果NEW状态的conntrack保留时间过短，就会导致大量NEW状态的连接，而对于很多依赖ctstate的模块而言，这样就会有问题，比如iptables的filter表中使用ESTABLISH状态来放过前向包的返回包就会有问题，此时ip_conntrack很有可能由于NEW状态时间过短而将返回包作为NEW状态处理而不是ESTABLISH状态，如此一来，返回包就无法通过了。如下图所示：

```
for(;;)
{
n = recvfrom(sd, msg, MAXLINE, 0, pcliaddr, &len);
sleep(5);
sendto(sd, msg, n, 0, pcliaddr, len);
}
```


然后在客户端上执行echo $sec /proc/sys/net/ipv4/netfilter/ip_conntrack_udp_timeout

其中sec要比服务器端的sleep参数更小即可。

如此udp客户端将收不到服务器eho回来的字符串，因为客户端只是放行状态为establish的入流量，如果ip_conntrack_udp_timeout配置过于短暂，NEW状态的conntrack过早被释放，这样将不会有establish状态的流量了。对于UDP而言，由于它是不确认无连接允许丢包的，因此影响还不是很大，TCP也有类似的问题，那就是如果你连接一个很远的且网络状况很恶劣的TCP服务器，然后你把ip_conntrack_tcp_timeout_synsent设置很小，这样就几乎完不成三次握手了，更进一步，如果你把ip_conntrack_tcp_timeout_established设置过小，那么一旦三次握手建立连接之后，客户端和服务器之间很久不发包，当establish状态到期后，conntrack被释放，此时服务器端主动发来一个包，该包的conntrack状态会是什么呢？因此给予tcp的establish状态5天的时间，是可以理解的。需要注意的是，对于tcp而言，由于无法简单的控制服务器发送syn-ack的延时，因此需要在establish状态而不是new状态做文章了(实际上，ip_conntrack的establish状态映射成了tcp的多个状态，包括syn-ack，ack，established)，试试看，效果和udp的一样。

前面关于ip_conntrack扯的太远了，我们的首要问题是conntrack full的问题。实际上，如果深入思考这个conntrack full的问题，就会发现，并不是conntrack容量太小或者表项保留时间过长引发的full。现实中的万事万物都不是无限的，对于计算机资源而言，更应该节约使用，不能让无关人士浪费这种资源，另外既然内核默认了一个表项的存活时间，那肯定是经过测试的经验值，自有它的道理。因此本质问题在于很多不需要conntrack的包也被conntrack了，这样就会挤掉很多真正需要conntrack的流量。

那么都是哪些流量需要conntrack呢？常用的就两个，一个是任何使用ctstate或者state这些match的iptables规则，另外一个就是所有的iptables的nat表中的规则，如果我们事先知道哪些流量需要使用iptables的[ct]state来控制，并且也知道哪些流量需要做NAT，那么余下的流量就都是和conntrack无关的流量了，可以不被ip_conntrack来跟踪。

幸运的是，Linux的Netfilter在PREROUTING以及OUTPUT这两个HOOK的conntrack之前安插了一个优先级更高的table，那就是raw，通过它就可以分离出不需要被conntrack的流量。如果你确定只有某个网卡进来的流量才需要做NAT，那么就执行下面的规则：

```
iptables -t raw -A PREROUTING ! –I $网卡 -j NOTRACK
iptables –t raw –A OUTPUT –j NOTRACK
```


这样一来，资源就不会浪费在无关人士身上了，性能也会有所提高，因为凡是NOTRACK的流量，都不会去查询conntrack的hash表，因为在ip(nf)_conntrack_in的内部的开始有一个判断：```
if ((*pskb)->nfct)
return NF_ACCEPT;
```


而NOTRACK这个target的实现也很简单：`(*pskb)->nfct = &ip_conntrack_untracked.info[IP_CT_NEW];`


事实上将一个占位者设置给skb的nfct，这样可以保持其它代码的一致性。可见，必要时同时采取三种方式比较有效：1.增大conntrack_max;2.减少状态保存时间;3.分离无关流量。然而除了第三种方式，其余两种方式在操作时必须给自己十足的理由那么做才行，对于1，比必须明白内核内存被占有的方式，对于2，看看本文的前半部分。

`iptables -A FORWARD -m state --state UNTRACKED -j ACCEPT`