---
title: Linux的TUN/TAP编程
source: https://blog.csdn.net/bailyzheng/article/details/18770121
kind: external
domain: network
author: 成就一亿技术人
original_date: 2014-08-04
fetched_at: 2026-05-16
bookmark_title: Linux的TUN/TAP编程 - 海纳百川 - CSDN博客
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/bailyzheng/article/details/18770121)
> 作者：成就一亿技术人
> 原始日期：2014-08-04
> 抓取日期：2026-05-16

# Linux的TUN/TAP编程

原理简介

TUN/TAP 虚拟网络设备的原理比较简单，他在Linux内核中添加了一个TUN/TAP虚拟网络设备的驱动程序和一个与之相关连的字符设备 /dev/net/tun，字符设备tun作为用户空间和内核空间交换数据的接口。当内核将数据包发送到虚拟网络设备时，数据包被保存在设备相关的一个队 列中，直到用户空间程序通过打开的字符设备tun的描述符读取时，它才会被拷贝到用户空间的缓冲区中，其效果就相当于，数据包直接发送到了用户空间。通过 系统调用write发送数据包时其原理与此类似。

值得注意的是：一次read系统调用，有且只有一个数据包被传送到用户空间，并且当用户空间的缓冲区比较小时，数据包将被截断，剩余部分将永久地消失，write系统调用与read类似，每次只发送一个数据包。所以在编写此类程序的时候，请用足够大的缓冲区，直接调用系统调用read/write，避免采用C语言的带缓存的IO函数。

准备工作

首先你需要一个能工作的Linux操作系统，并且内核支持TUN/TAP虚拟网络设备，如果没有，请在内核中选中：

Device Drivers => Network device support => Universal TUN/TAP device driver support

你可以选择编译进内核或者是编译成模块，然后重新编译内核并用新内核启动。如果你编译的是模块，那么在下步开始之前，你需要手工加载它。

root@gentux ~ # modprobe tun

开始编程

从代码开始，

`12 #include <linux/if_tun.h>`


13

14 int tun_create(char *dev, int flags)

15 {

16 struct ifreq ifr;

17 int fd, err;

18

19 assert(dev != NULL);

20

21 if ((fd = open("/dev/net/tun", O_RDWR)) < 0)

22 return fd;

23

24 memset(&ifr, 0, sizeof(ifr));

25 ifr.ifr_flags |= flags;

26 if (*dev != '\0')

27 strncpy(ifr.ifr_name, dev, IFNAMSIZ);

28 if ((err = ioctl(fd, TUNSETIFF, (void *)&ifr)) < 0) {

29 close(fd);

30 return err;

31 }

32 strcpy(dev, ifr.ifr_name);

33

34 return fd;

35 }


为了使用TUN/TAP设备，我们必须包含特定的头 文件linux/if_tun.h，如12行所示。在21行，我们打开了字符设备/dev/net/tun。接下来我们需要为ioctl的 TUNSETIFF命令初始化一个结构体ifr，一般的时候我们只需要关心其中的两个成员ifr_name, ifr_flags。ifr_name定义了要创建或者是打开的虚拟网络设备的名字，如果它为空或者是此网络设备不存在，内核将新建一个虚拟网络设备，并 返回新建的虚拟网络设备的名字，同时文件描述符fd也将和此网络设备建立起关联。如果并没有指定网络设备的名字，内核将根据其类型自动选择tunXX和 tapXX作为其名字。ifr_flags用来描述网络设备的一些属性，比如说是点对点设备还是以太网设备。详细的选项解释如下:

- IFF_TUN: 创建一个点对点设备
- IFF_TAP: 创建一个以太网设备
- IFF_NO_PI: 不包含包信息，默认的每个数据包当传到用户空间时，都将包含一个附加的包头来保存包信息
- IFF_ONE_QUEUE: 采用单一队列模式，即当数据包队列满的时候，由虚拟网络设备自已丢弃以后的数据包直到数据包队列再有空闲。

`struct tun_pi {`


unsigned short flags;

unsigned short proto;

};


目前，flags只在收取数据包的时候有效，当它的TUN_PKT_STRIP标志被置时，表示当前的用户空间缓冲区太小，以致数据包被截断。proto成员表示发送/接收的数据包的协议。

上面代码中的文件描述符fd除了支持TUN_SETIFF和其他的常规ioctl命令外，还支持以下命令:

- TUNSETNOCSUM: 不做校验和校验。参数为int型的bool值。
- TUNSETPERSIST: 把对应网络设备设置成持续模式，默认的虚拟网络设备，当其相关的文件符被关闭时，也将会伴随着与之相关的路由等信息同时消失。如果设置成持续模式，那么它将会被保留供以后使用。参数为int型的bool值。
- TUNSETOWNER: 设置网络设备的属主。参数类型为uid_t。
- TUNSETLINK: 设置网络设备的链路类型，此命令只有在虚拟网络设备关闭的情况下有效。参数为int型。

`int main(int argc, char *argv[])`


{

int tun, ret;

char tun_name[IFNAMSIZ];

unsigned char buf[4096];


tun_name[0] = '\0';

tun = tun_create(tun_name, IFF_TUN | IFF_NO_PI);

if (tun < 0) {

perror("tun_create");

return 1;

}

printf("TUN name is %s\n", tun_name);


while (1) {

unsigned char ip[4];


ret = read(tun, buf, sizeof(buf));

if (ret < 0)

break;

memcpy(ip, &buf[12], 4);

memcpy(&buf[12], &buf[16], 4);

memcpy(&buf[16], ip, 4);

buf[20] = 0;

*((unsigned short*)&buf[22]) += 8;

printf("read %d bytes\n", ret);

ret = write(tun, buf, ret);

printf("write %d bytes\n", ret);

}


return 0;

}


以上代码简答地处理了ICMP的ECHO包，并回应以ECHO REPLY。

首先运行这个程序:

root@gentux test # ./a.out

TUN name is tun0

接着在另外一个终端运行如下命令:

root@gentux linux-2.6.15-gentoo # ifconfig tun0 0.0.0.0 up

root@gentux linux-2.6.15-gentoo # route add 10.10.10.1 dev tun0

root@gentux linux-2.6.15-gentoo # ping 10.10.10.1

PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.

64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.09 ms

64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=5.18 ms

64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=3.37 ms

--- 10.10.10.1 ping statistics ---

3 packets transmitted, 3 received, 0% packet loss, time 2011ms

rtt min/avg/max/mdev = 1.097/3.218/5.181/1.671 ms

可见，我们顺利地接受到了回应包，这时，切回到前一个终端下:

read 84 bytes

write 84 bytes

read 84 bytes

write 84 bytes

read 84 bytes

write 84 bytes

一切正如我们所预想的那样。

TUN/TAP能做什么？

hoho，问这个问题似乎有些傻，你说一个网卡能做什么？我可以告诉你两个基于此的开源项目：vtun和openvpn，至于其他的应用，请自由发挥你的想像力吧！




## 虚拟网卡 TUN/TAP 驱动程序设计原理

虚拟网卡Tun/tap驱动是一个开源项目，支持很多的类UNIX平台，OpenVPN和Vtun都是基于它实现隧道包封装。本文将介绍tun/tap驱动的使用并分析虚拟网卡tun/tap驱动程序在linux环境下的设计思路。

tun/tap驱动程序实现了虚拟网卡的功能，tun表示虚拟的是点对点设备，tap表示虚拟的是以太网设备，这两种设备针对网络包实施不同的封装。利用tun/tap驱动，可以将tcp/ip协议栈处理好的网络分包传给任何一个使用tun/tap驱动的进程，由进程重新处理后再发到物理链路中。开源项目openvpn（ http://openvpn.sourceforge.net）和Vtun( http://vtun.sourceforge.net)都是利用tun/tap驱动实现的隧道封装。

在linux 2.4内核版本及以后版本中，tun/tap驱动是作为系统默认预先编译进内核中的。在使用之前，确保已经装载了tun/tap模块并建立设备文件：

#modprobe tun #mknod /dev/net/tun c 10 200 |

参数c表示是字符设备, 10和200分别是主设备号和次设备号。

这样，我们就可以在程序中使用该驱动了。

使用tun/tap设备的示例程序(摘自openvpn开源项目 http://openvpn.sourceforge.net，tun.c文件)

int open_tun (const char *dev, char *actual, int size) { struct ifreq ifr; int fd; char *device = "/dev/net/tun"; if ((fd = open (device, O_RDWR)) < 0) //创建描述符 msg (M_ERR, "Cannot open TUN/TAP dev %s", device); memset (&ifr, 0, sizeof (ifr)); ifr.ifr_flags = IFF_NO_PI; if (!strncmp (dev, "tun", 3)) { ifr.ifr_flags |= IFF_TUN; } else if (!strncmp (dev, "tap", 3)) { ifr.ifr_flags |= IFF_TAP; } else { msg (M_FATAL, "I don't recognize device %s as a TUN or TAP device",dev); } if (strlen (dev) > 3) /* unit number specified? */ strncpy (ifr.ifr_name, dev, IFNAMSIZ); if (ioctl (fd, TUNSETIFF, (void *) &ifr) < 0) //打开虚拟网卡 msg (M_ERR, "Cannot ioctl TUNSETIFF %s", dev); set_nonblock (fd); msg (M_INFO, "TUN/TAP device %s opened", ifr.ifr_name); strncpynt (actual, ifr.ifr_name, size); return fd; } |

调用上述函数后，就可以在shell命令行下使用ifconfig 命令配置虚拟网卡了，通过生成的字符设备描述符，在程序中使用read和write函数就可以读取或者发送给虚拟的网卡数据了。

做为虚拟网卡驱动，Tun/tap驱动程序的数据接收和发送并不直接和真实网卡打交道，而是通过用户态来转交。在linux下，要实现核心态和用户态数据的交互，有多种方式：可以通用socket创建特殊套接字，利用套接字实现数据交互；通过proc文件系统创建文件来进行数据交互；还可以使用设备文件的方式，访问设备文件会调用设备驱动相应的例程，设备驱动本身就是核心态和用户态的一个接口，Tun/tap驱动就是利用设备文件实现用户态和核心态的数据交互。

从结构上来说，Tun/tap驱动并不单纯是实现网卡驱动，同时它还实现了字符设备驱动部分。以字符设备的方式连接用户态和核心态。下面是示意图：

Tun/tap驱动程序中包含两个部分，一部分是字符设备驱动，还有一部分是网卡驱动部分。利用网卡驱动部分接收来自TCP/IP协议栈的网络分包并发送或者反过来将接收到的网络分包传给协议栈处理，而字符驱动部分则将网络分包在内核与用户态之间传送，模拟物理链路的数据接收和发送。Tun/tap驱动很好的实现了两种驱动的结合。

**下面是定义的tun/tap设备结构：**

struct tun_struct { char name[8]; //设备名 unsigned long flags; //区分tun和tap设备 struct fasync_struct *fasync; //文件异步通知结构 wait_queue_head_t read_wait; //等待队列 struct net_device dev; //linux 抽象网络设备结构 struct sk_buff_head txq; //网络缓冲区队列 struct net_device_stats stats; //网卡状态信息结构 }; |

struct net_device结构是linux内核提供的统一网络设备结构，定义了系统统一的访问接口。

**Tun/tap驱动中实现的网卡驱动的处理例程：**

static int tun_net_open(struct net_device *dev)；

static int tun_net_close(struct net_device *dev)；

static int tun_net_xmit(struct sk_buff *skb, struct net_device *dev)；//数据包发送例程

static void tun_net_mclist(struct net_device *dev)；//设置多点传输的地址链表

static struct net_device_stats *tun_net_stats(struct net_device *dev)；//当一个应用程序需要知道 网络接口的一些统计数据时，可调用该函数，如ifconfig、netstat等。

int tun_net_init(struct net_device *dev)；//网络设备初始例程

**字符设备部分：**

在Linux中，字符设备和块设备统一以文件的方式访问，访问它们的接口是统一的，都是使用open()函数打开设备文件或普通文件，用read()和write()函数实现读写文件等等。Tun/tap驱动定义的字符设备的访问接口如下：

static struct file_operations tun_fops = {

owner: THIS_MODULE,

llseek: tun_chr_lseek,

read tun_chr_read,

write: tun_chr_write,

poll: tun_chr_poll,

ioctl: tun_chr_ioctl,

open: tun_chr_open,

release: tun_chr_close,

fasync: tun_chr_fasync

};

在内核中利用misc_register() 函数将该驱动注册为非标准字符设备驱动，提供字符设备具有的各种程序接口。代码摘自linux-2.4.20\linux-2.4.20\drivers\net\tun.c

static struct miscdevice tun_miscdev= { TUN_MINOR, "net/tun", &tun_fops }; int __init tun_init(void) { … if (misc_register(&tun_miscdev)) { printk(KERN_ERR "tun: Can't register misc device %d\n", TUN_MINOR); return -EIO; } return 0; } |

当打开一个tun/tap设备时，open 函数将调用tun_chr_open()函数，其中将完成一些重要的初始化过程，包括设置网卡驱动部分的初始化函数以及网络缓冲区链表的初始化和等待队列的初始化。Tun/tap驱动中网卡的注册被嵌入了字符驱动的ioctl例程中，它是通过对字符设备文件描述符利用自定义的ioctl设置标志TUNSETIFF完成网卡的注册的。下面是函数调用关系示意图：

使用ioctl()函数操作字符设备文件描述符,将调用字符设备中tun_chr_ioctl 来设置已经open好的tun/tap设备，如果设置标志为TUNSETIFF，则调用tun_set_iff() 函数，此函数将完成很重要的一步操作，就是对网卡驱动进行注册register_netdev(&tun->dev)，网卡驱动的各个处理例程的挂接在open操作时由tun_chr_open()函数初始化好了。

**Tun/tap设备的工作过程：**

Tun/tap设备提供的虚拟网卡驱动，从tcp/ip协议栈的角度而言，它与真实网卡驱动并没有区别。从驱动程序的角度来说，它与真实网卡的不同表现在tun/tap设备获取的数据不是来自物理链路，而是来自用户区，Tun/tap设备驱动通过字符设备文件来实现数据从用户区的获取。发送数据时tun/tap设备也不是发送到物理链路，而是通过字符设备发送至用户区，再由用户区程序通过其他渠道发送。

**发送过程：**

使用tun/tap网卡的程序经过协议栈把数据传送给驱动程序，驱动程序调用注册好的hard_start_xmit函数发送，hard_start_xmit函数又会调用tun_net_xmit函数，其中skb将会被加入skb链表，然后唤醒被阻塞的使用tun/tap设备字符驱动读数据的进程，接着tun/tap设备的字符驱动部分调用其tun_chr_read()过程读取skb链表，并将每一个读到的skb发往用户区，完成虚拟网卡的数据发送。

**接收数据的过程：**

当我们使用write()系统调用向tun/tap设备的字符设备文件写入数据时，tun_chr_write函数将被调用，它使用tun_get_user从用户区接受数据，其中将数据存入skb中，然后调用关键的函数netif_rx(skb) 将skb送给tcp/ip协议栈处理，完成虚拟网卡的数据接收。

tun/tap驱动很巧妙的将字符驱动和网卡驱动糅合在一起，本文重点分析两种驱动之间的衔接，具体的驱动处理细节没有一一列出，请读者参考相关文档。

- 《Linux 设备驱动程序 第二版》，(美)Alessando Rubini


- http://openvpn.sourceforge.net


- Linux串口上网的简单实现


- linux 源代码