---
title: 010 使用netmap API接管网卡，接收数据包，回应ARP请求
source: https://www.cnblogs.com/ruo-yu/p/5084270.html
kind: external
domain: network
author: Ruo Yu
original_date: 2015-12-29
fetched_at: 2026-05-16
bookmark_title: 010 使用netmap API接管网卡，接收数据包，回应ARP请求 - ruo_yu - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/ruo-yu/p/5084270.html)
> 作者：Ruo Yu
> 原始日期：2015-12-29
> 抓取日期：2026-05-16

# 010 使用netmap API接管网卡，接收数据包，回应ARP请求

# 010 使用netmap API接管网卡，接收数据包，回应ARP请求

**一.本文目的：**

上一节中，我们已经在CentOS 6.7 上安装好了netmap，也能接收和发送包了，这节我们来调用netmap中的API，接管网卡，对网卡上收到的数据包做分析，并回应ARP请求。


**二.netmap API简要介绍：**

1.netmap API 主要包含在两个头文件中：netmap.h和netmap_user.h。在netmap/sys/net/目录下，其中netmap_user.h调用netmap.h。

2.netmap API一共七八个函数调用：nm_open()生成文件描述符，并做一系列初始化操作。nm_mmap()被nm_open()调用，申请存放数据包的内存池，并做相应初始化。

3.nm_nexkpkt()函数负责取出内存池的数据包，nm_inject()函数往内存池中写入数据包，发送到网卡。

4.nm_close()函数关闭先前的有关操作，并做相应清理。

（本文的目的是对ARP数据报的接收发送分析，所以对netmap API先只是简单介绍。）


**三.具体内容：**

1.实验中主机为centos 6.7,虚拟机也为centos6.7.所以就直接用主机给虚拟机发udp数据包了。(单纯的网络环境，没有其它主机的干扰！)

2.当调用了netmap API的程序运行时，会接管网卡，网卡上的数据都要通过netmap API中的方法得到(包括发送)。

3.当我们拿到数据包的时候，是一个含以太网首部的完整数据包，我们需要查看数据包的结构，确认是发给自己的。

4.实验发现，当网卡被接管后，相应的ARP功能没有了，所以我们需要手动实现一个ARP reply程序。


**四.数据结构的定义：**

实验过程中，会收到ARP请求和UDP数据包，我们主要对这两者进行分析：

**1.完整的以太网udp数据包结构**

结构体定义：

struct udp_pkt /* 普通的完整udp数据包 */ { struct ether_header eh; /* 以太网头部,14字节,<net/ethernet.h>头文件中 */ struct iphdr ip; /* ip部分,20字节,<netinet/ip.h>头文件中 */ struct udphdr udp; /* udp部分,8字节,<netinet/udp.h>头文件中 */ uint8_t body[20]; /* 数据部分,暂时分配20字节 */ };


**2.完整的以太网ARP数据包结构**

ARP数据报结构图：


结构体定义：

struct arp_pkt /* 完整arp数据包(以太网首部 + ARP字段) */ { struct ether_header eh; /* 以太网头部,14字节,<net/ethernet.h>头文件中 */ struct ether_arp arp; /* ARP字段 ,28字节,<netinet/if_ether.h>头文件中 */ };


**五.相关的函数封装(以后使用)**

1.打印mac地址函数：

void Print_mac_addr(const unsigned char *str) /* 打印mac地址 */ { int i; for (i = 0; i < 5; i++) printf("%02x:", str[i]); printf("%02x", str[i]); }

2.打印ip地址：

void Print_ip_addr(const unsigned char *str) /* 打印ip地址 */ { int i; for (i = 0; i < 3; i++) printf("%d.", str[i]); printf("%d", str[i]); }

3.打印完整的arp数据包的内容

void Print_arp_pkt(struct arp_pkt* arp_pkt) /* 打印完整的以太网arp数据包的内容 */ { Print_mac_addr(arp_pkt->eh.ether_dhost), printf(" "); /* 以太网目的地址 */ Print_mac_addr(arp_pkt->eh.ether_shost), printf(" "); /* 以太网源地址 */ printf("0x%04x ", ntohs(arp_pkt->eh.ether_type)); /* 帧类型:0x0806 */ printf(" "); printf("%d ", ntohs(arp_pkt->arp.ea_hdr.ar_hrd)); /* 硬件类型:1 */ printf("0x%04x ", ntohs(arp_pkt->arp.ea_hdr.ar_pro)); /* 协议类型:0x0800 */ printf("%d ",arp_pkt->arp.ea_hdr.ar_hln); /* 硬件地址:6 */ printf("%d ",arp_pkt->arp.ea_hdr.ar_pln); /* 协议地址长度:4 */ printf("%d ", ntohs(arp_pkt->arp.ea_hdr.ar_op)); /* 操作字段:ARP请求值为1,ARP应答值为2 */ printf(" "); Print_mac_addr(arp_pkt->arp.arp_sha), printf(" "); /* 发送端以太网地址*/ Print_ip_addr(arp_pkt->arp.arp_spa), printf(" "); /* 发送端IP地址 */ Print_mac_addr(arp_pkt->arp.arp_tha), printf(" "); /* 目的以太网地址 */ Print_ip_addr(arp_pkt->arp.arp_tpa), printf(" "); /* 目的IP地址 */ printf("\n"); }

4.根据ARP request生成ARP reply的packet

/* * 根据ARP request生成ARP reply的packet * hmac为本机mac地址 */ void Init_echo_pkt(struct arp_pkt *arp, struct arp_pkt *arp_rt, char *hmac) { bcopy(arp->eh.ether_shost, arp_rt->eh.ether_dhost, 6); /* 填入目的地址 */ bcopy(ether_aton(hmac), arp_rt->eh.ether_shost, 6); /* hmac为本机mac地址 */ arp_rt->eh.ether_type = arp->eh.ether_type; /* 以太网帧类型 */ ; arp_rt->arp.ea_hdr.ar_hrd = arp->arp.ea_hdr.ar_hrd; arp_rt->arp.ea_hdr.ar_pro = arp->arp.ea_hdr.ar_pro; arp_rt->arp.ea_hdr.ar_hln = 6; arp_rt->arp.ea_hdr.ar_pln = 4; arp_rt->arp.ea_hdr.ar_op = htons(2); /* ARP应答 */ ; bcopy(ether_aton(hmac), &arp_rt->arp.arp_sha, 6); /* 发送端以太网地址*/ bcopy(arp->arp.arp_tpa, &arp_rt->arp.arp_spa, 4); /* 发送端IP地址 */ bcopy(arp->arp.arp_sha, &arp_rt->arp.arp_tha, 6); /* 目的以太网地址 */ bcopy(arp->arp.arp_spa, &arp_rt->arp.arp_tpa, 4); /* 目的IP地址 */ }

**六.源码实现：**

其中：

本机mac地址：00:0C:29:E4:D6:2A

本机ip地址：192.168.11.134

发送udp数据包程序：

1 /* 2 ============================================================================ 3 Name : send_packet_01.c 4 Author : huh 5 Version : 6 Copyright : huh's copyright notice 7 Description : Hello World in C, Ansi-style 8 ============================================================================ 9 */ 10 11 #include <sys/types.h> 12 #include <sys/socket.h> 13 #include <stdio.h> 14 #include <netinet/in.h> 15 #include <arpa/inet.h> 16 #include <unistd.h> 17 #include <stdlib.h> 18 #include <string.h> 19 #include <netinet/ip_icmp.h> 20 #include <netinet/udp.h> 21 22 #define MAXLINE 1024*10 23 24 int main() 25 { 26 int sockfd; 27 struct sockaddr_in server_addr; 28 //创建原始套接字 29 sockfd = socket(AF_INET, SOCK_DGRAM, 0); 30 //创建套接字地址 31 bzero(&server_addr,sizeof(server_addr)); 32 server_addr.sin_family = AF_INET; 33 server_addr.sin_port = htons(8686); 34 server_addr.sin_addr.s_addr = inet_addr("192.168.11.134"); 35 char sendline[10]="adcde"; 36 while(1) 37 { 38 sendto(sockfd, sendline, strlen(sendline), 0, (struct sockaddr *)&server_addr, sizeof(server_addr)); 39 printf("%s\n",sendline); 40 sleep(1); 41 } 42 return 0; 43 }

接收数据包程序：

1 /* 2 ============================================================================ 3 Name : recv_pkt.c 4 Author : huh 5 Version : 6 Copyright : huh's copyright notice 7 Description : Hello World in C, Ansi-style 8 ============================================================================ 9 */ 10 11 #include <stdio.h> 12 #include <unistd.h> 13 #include <stdlib.h> 14 #include <string.h> 15 #include <ifaddrs.h> 16 #include <net/ethernet.h> 17 #include <netinet/in.h> 18 #include <netinet/ip.h> 19 #include <netinet/udp.h> 20 #include <netinet/ether.h> 21 #include <netinet/if_ether.h> 22 23 #include <unistd.h> // sysconf() 24 #include <sys/poll.h> 25 #include <arpa/inet.h> 26 27 #include "netmap_user.h" /* 来源与netmap */ 28 #pragma pack(1) /* 设置结构体的边界对齐为1个字节 */ 29 30 struct udp_pkt /* 普通的完整udp数据包 */ 31 { 32 struct ether_header eh; /* 以太网头部,14字节,<net/ethernet.h>头文件中 */ 33 struct iphdr ip; /* ip部分,20字节,<netinet/ip.h>头文件中 */ 34 struct udphdr udp; /* udp部分,8字节,<netinet/udp.h>头文件中 */ 35 uint8_t body[20]; /* 数据部分,暂时分配20字节 */ 36 }; 37 38 struct arp_pkt /* 完整arp数据包(以太网首部 + ARP字段) */ 39 { 40 struct ether_header eh; /* 以太网头部,14字节,<net/ethernet.h>头文件中 */ 41 struct ether_arp arp; /* ARP字段 ,28字节,<netinet/if_ether.h>头文件中 */ 42 }; 43 44 void Print_mac_addr(const unsigned char *str) /* 打印mac地址 */ 45 { 46 int i; 47 for (i = 0; i < 5; i++) 48 printf("%02x:", str[i]); 49 printf("%02x", str[i]); 50 } 51 52 void Print_ip_addr(const unsigned char *str) /* 打印ip地址 */ 53 { 54 int i; 55 for (i = 0; i < 3; i++) 56 printf("%d.", str[i]); 57 printf("%d", str[i]); 58 } 59 60 void Print_arp_pkt(struct arp_pkt* arp_pkt) /* 打印完整的arp数据包的内容 */ 61 { 62 Print_mac_addr(arp_pkt->eh.ether_dhost), printf(" "); /* 以太网目的地址 */ 63 Print_mac_addr(arp_pkt->eh.ether_shost), printf(" "); /* 以太网源地址 */ 64 printf("0x%04x ", ntohs(arp_pkt->eh.ether_type)); /* 帧类型:0x0806 */ 65 printf(" "); 66 printf("%d ", ntohs(arp_pkt->arp.ea_hdr.ar_hrd)); /* 硬件类型:1 */ 67 printf("0x%04x ", ntohs(arp_pkt->arp.ea_hdr.ar_pro)); /* 协议类型:0x0800 */ 68 printf("%d ",arp_pkt->arp.ea_hdr.ar_hln); /* 硬件地址:6 */ 69 printf("%d ",arp_pkt->arp.ea_hdr.ar_pln); /* 协议地址长度:4 */ 70 printf("%d ", ntohs(arp_pkt->arp.ea_hdr.ar_op)); /* 操作字段:ARP请求值为1,ARP应答值为2 */ 71 printf(" "); 72 Print_mac_addr(arp_pkt->arp.arp_sha), printf(" "); /* 发送端以太网地址*/ 73 Print_ip_addr(arp_pkt->arp.arp_spa), printf(" "); /* 发送端IP地址 */ 74 Print_mac_addr(arp_pkt->arp.arp_tha), printf(" "); /* 目的以太网地址 */ 75 Print_ip_addr(arp_pkt->arp.arp_tpa), printf(" "); /* 目的IP地址 */ 76 printf("\n"); 77 } 78 79 /* 80 * 根据ARP request生成ARP reply的packet 81 * hmac为本机mac地址 82 */ 83 void Init_echo_pkt(struct arp_pkt *arp, struct arp_pkt *arp_rt, char *hmac) 84 { 85 bcopy(arp->eh.ether_shost, arp_rt->eh.ether_dhost, 6); /* 填入目的地址 */ 86 bcopy(ether_aton(hmac), arp_rt->eh.ether_shost, 6); /* hmac为本机mac地址 */ 87 arp_rt->eh.ether_type = arp->eh.ether_type; /* 以太网帧类型 */ 88 ; 89 arp_rt->arp.ea_hdr.ar_hrd = arp->arp.ea_hdr.ar_hrd; 90 arp_rt->arp.ea_hdr.ar_pro = arp->arp.ea_hdr.ar_pro; 91 arp_rt->arp.ea_hdr.ar_hln = 6; 92 arp_rt->arp.ea_hdr.ar_pln = 4; 93 arp_rt->arp.ea_hdr.ar_op = htons(2); /* ARP应答 */ 94 ; 95 bcopy(ether_aton(hmac), &arp_rt->arp.arp_sha, 6); /* 发送端以太网地址*/ 96 bcopy(arp->arp.arp_tpa, &arp_rt->arp.arp_spa, 4); /* 发送端IP地址 */ 97 bcopy(arp->arp.arp_sha, &arp_rt->arp.arp_tha, 6); /* 目的以太网地址 */ 98 bcopy(arp->arp.arp_spa, &arp_rt->arp.arp_tpa, 4); /* 目的IP地址 */ 99 } 100 101 int main() 102 { 103 struct ether_header *eh; 104 struct nm_pkthdr h; 105 struct nm_desc *nmr; 106 nmr = nm_open("netmap:eth0", NULL, 0, NULL); /* 打开netmap对应的文件描述符,并做了相关初始化操作！ */ 107 struct pollfd pfd; 108 pfd.fd = nmr->fd; 109 pfd.events = POLLIN; 110 u_char *str; 111 printf("ready!!!\n"); 112 while (1) 113 { 114 poll(&pfd, 1, -1); /* 使用poll来监听是否有事件到来 */ 115 if (pfd.revents & POLLIN) 116 { 117 str = nm_nextpkt(nmr, &h); /* 接收到来的数据包 */ 118 eh = (struct ether_header *) str; 119 if (ntohs(eh->ether_type) == 0x0800) /* 接受的是普通IP包 */ 120 { 121 struct udp_pkt *p = (struct udp_pkt *)str; 122 if(p->ip.protocol == IPPROTO_UDP) /*如果是udp协议的数据包*/ 123 printf("udp:%s\n", p->body); 124 else 125 printf("其它IP协议包!\n"); 126 } 127 else if (ntohs(eh->ether_type) == 0x0806) /* 接受的是ARP包 */ 128 { 129 struct arp_pkt arp_rt; 130 struct arp_pkt *arp = (struct arp_pkt *)str; 131 if( *(uint32_t *)arp->arp.arp_tpa == inet_addr("192.168.11.134") ) /*如果请求的IP是本机IP地址(两边都是网络字节序)*/ 132 { 133 printf("ARP请求:"); 134 Print_arp_pkt(arp); 135 Init_echo_pkt(arp, &arp_rt, "00:0C:29:E4:D6:2A"); /*根据收到的ARP请求生成ARP应答数据包*/ 136 printf("ARP应答:"); 137 Print_arp_pkt(&arp_rt); 138 nm_inject(nmr, &arp_rt, sizeof(struct arp_pkt)); /* 发送ARP应答包 */ 139 } 140 else 141 printf("其它主机的ARP请求！\n"); 142 } 143 } 144 } 145 nm_close(nmr); 146 return 0; 147 }


**七.运行结果:**

```
ready!!!
udp:adcde
udp:adcde
udp:adcde
udp:adcde
udp:adcde
udp:adcde
udp:adcde
udp:adcde
udp:adcde
udp:adcde
ARP请求:00:0c:29:e4:d6:2a 00:50:56:c0:00:08 0x0806 1 0x0800 6 4 1 00:50:56:c0:00:08 192.168.11.1 00:00:00:00:00:00 192.168.11.134
ARP应答:00:50:56:c0:00:08 00:0c:29:e4:d6:2a 0x0806 1 0x0800 6 4 2 00:0c:29:e4:d6:2a 192.168.11.134 00:50:56:c0:00:08 192.168.11.1
udp:adcde
udp:adcde
udp:adcde
```