---
title: TCP的三次握手与四次分手
source: https://www.cnblogs.com/leezhxing/p/4524176.html
kind: external
domain: network
author: Leezhxing
original_date: 2015-05-23
fetched_at: 2026-05-16
bookmark_title: TCP的三次握手与四次分手 - leezhxing - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/leezhxing/p/4524176.html)
> 作者：Leezhxing
> 原始日期：2015-05-23
> 抓取日期：2026-05-16

# TCP的三次握手与四次分手

# TCP的三次握手与四次分手

**TCP的位置**

TCP工作在网络OSI的七层模型中的第四层——Transport层，IP在第三层——Network层，ARP在第二层——Data Link层；

在第二层上的数据，我们把它叫Frame，在第三层上的数据叫Packet，第四层的数据叫Segment。

数据从应用层发下来，会在每一层都会加上头部信息，进行封装，然后再发送到数据接收端。这个基本的流程你需要知道，就是每个数据都会经过数据的封装和解封装的过程。 在OSI七层模型中，每一层的作用和对应的协议如下：


**3次握手**

第一次握手：主机A发送位码为syn＝1,随机产生seq number=x的数据包到服务器，客户端进入`SYN_SEND`

状态，等待服务器的确认；主机B由SYN=1知道，A要求建立联机；

第二次握手：主机B收到请求后要确认联机信息，向A发送ack number(主机A的seq+1),syn=1,ack=1,随机产生seq=y的包,此时服务器进入`SYN_RECV`

状态;

第三次握手：主机A收到后检查ack number是否正确，即第一次发送的seq number+1,以及位码ack是否为1，若正确，主机A会再发送ack number(主机B的seq+1),ack=1，主机B收到后确认seq值与ack=1则连接建立成功。客户端和服务器端都进入`ESTABLISHED`

状

态，完成TCP三次握手。

TCP位码,有6种标示:SYN(synchronous建立联机) ACK(acknowledgement 确认) PSH(push传送) FIN(finish结束) RST(reset重置) URG(urgent紧急)

Sequence number(顺序号码) Acknowledge number(确认号码)



**4次挥手**

第一次挥手：主机1（可以使客户端，也可以是服务器端），设置`Sequence Number`

和`Acknowledgment Number`

，向主机2发送一个`FIN`

报文段；此时，主机1进入`FIN_WAIT_1`

状态；这表示主机1没有数据要发送给主机2了；

第二次挥手：主机2收到了主机1发送的`FIN`

报文段，向主机1回一个`ACK`

报文段，`Acknowledgment Number`

为`Sequence Number`

加1；主机1进入`FIN_WAIT_2`

状态；主机2告诉主机1，我也没有数据要发送了，可以进行关闭连接了；

第三次挥手：主机2向主机1发送`FIN`

报文段，请求关闭连接，同时主机2进入`CLOSE_WAIT`

状态；

第四次挥手：主机1收到主机2发送的`FIN`

报文段，向主机2发送`ACK`

报文段，然后主机1进入`TIME_WAIT`

状态；主机2收到主机1的`ACK`

报文段以后，就关闭连接；此时，主机1等待2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，主机1也可以关闭连接了。



** 问题**

1.为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？

虽然按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假象网络是不可靠的，有可以最后一个ACK丢失。所以TIME_WAIT状态就是用来重发可能丢失的ACK报文。

2.client发送完最后一个ack之后，进入time_wait状态，但是他怎么知道server有没有收到这个ack呢？莫非sever也要等待一段时间，如果收到了这个ack就close，如果没有收到就再发一个fin给client？这么说server最后也有一个time_wait哦？求解答！

因为网络原因，主动关闭的一方发送的这个ACK包很可能延迟，从而触发被动连接一方重传FIN包。极端情况下，这一去一回，就是两倍的MSL时长。如果主动关闭的一方跳过TIME_WAIT直接进入CLOSED，或者在TIME_WAIT停留的时长不足两倍的MSL，那么当被动

关闭的一方早先发出的延迟包到达后，就可能出现类似下面的问题：1.旧的TCP连接已经不存在了，系统此时只能返回RST包2.新的TCP连接被建立起来了，延迟包可能干扰新的连接，这就是为什么time_wait需要等待2MSL时长的原因。


收集整理自：（果冻想还有TCP首部，TCP Flags，3次握手4次分手原因详细解释，必看必看）