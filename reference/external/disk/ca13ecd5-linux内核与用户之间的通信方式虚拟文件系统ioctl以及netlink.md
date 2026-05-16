---
title: linux内核与用户之间的通信方式——虚拟文件系统、ioctl以及netlink
source: https://blog.csdn.net/dragon_li_chen/article/details/8235060
kind: external
domain: disk
author: 成就一亿技术人
original_date: 2026-03-29
fetched_at: 2026-05-16
bookmark_title: linux内核与用户之间的通信方式——虚拟文件系统、ioctl以及netlink - 陈立龙的专栏 - CSDN博客
tags: [external, disk]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/dragon_li_chen/article/details/8235060)
> 作者：成就一亿技术人
> 原始日期：2026-03-29
> 抓取日期：2026-05-16

# linux内核与用户之间的通信方式——虚拟文件系统、ioctl以及netlink

本文尝试去阐述内核与用户空间之间的通信接口：虚拟文件系统、ioctl以及netlink.文中所有的结构及代码全来自于**Linux kernel 2.6.34.**


一、虚拟文件系统

proc文件系统，通常是挂载在/proc，允许内核以文件类型形式向用户提供内部信息，但是值得注意的是里面的文件目录不能被写入，即用户不能添加或者删除目录中的任何目录。同时，内核也提供了一个可供用户配置内核变量的方法，那就是sysctl系统调用以及在/proc中添加一个特殊目录：sys，为每个由sysctl所输出的内核变量引入一个文件。其中procfs主要是输出只读数据，而大多数sysctl信息都是可写的。

procfs: 当用户需要读取proc目录中文件内容时，就会间接引起一组注册函数的运行，生成并返回需要显示的内容。proc中的目录由内核中的proc_mkdir函数创建，/proc/net中的文件可以在proc_fs.h中定义的proc_net_fops_create和proc_net_remove函数进行注册与删除。以arp为例，首先会在/proc/net目录下创建arp文件（只读），然后初始化其**文件操作处理函数**，具体如下：

```
static const struct file_operations arp_seq_fops = {
.open = arp_seq_open,
.read = seq_read,
.llseek = seq_lseek,
.release = seq_release_net,
.owner = THIS_MODULE
};
```


sysctl（目录/proc/sys）：/proc/sys目录下的每一个文件实质上是一个内核变量，这些文件的访问规则是：任何人均可读，但是只有超级用户才能修改，修改方法主要有两种：直接修改指定文件，或者可以直接用sysctl系统调用进行修改（当然也有权限限制了）。

sysctl net.ipv4.ip_forward

sysctl -w net.ipv4.ip_forward=1 //修改值

这些目录或者文件既有可能是在引导期间创建的，也有可能是在运行期间生成的，在运行期间生成的情况主要有：

1. 当内核模块实现一个新功能，或者一个协议加载或者卸载时；

2. 当一个新的网络设备被注册或者删除时，

/proc/sys下的目录或者文件均是以ctl_table结构定义的。而要想注册或者删除ctl_table结构则是通过register_sysctl_table和unregister_sysctl_table函数来实现（kernel/sysctl.c）。

```
struct ctl_table
{
const char *procname; //proc/sys中所用的文件名
void *data;
int maxlen; //输出的内核变量的尺寸大小
mode_t mode; //设置/proc/sys中相关联的文件或者目录的访问权限
struct ctl_table *child; //用于建立目录与文件之间的父子关系
struct ctl_table *parent; /* Automatically set */
proc_handler *proc_handler; //完成读取或者写入操作的函数
void *extra1;
void *extra2; //两个可选参数，通常用于定义变量的最小值和最大值ֵ
};
```


一般来讲，/proc/sys下定义了以下几个主目录(kernel, vm, fs, debug, dev)，其以及它的子目录定义如下：

```
static struct ctl_table root_table[] = {
{
.procname = "kernel",
.mode = 0555,
.child = kern_table,
},
{
.procname = "vm",
.mode = 0555,
.child = vm_table,
},
{
.procname = "fs",
.mode = 0555,
.child = fs_table,
},
{
.procname = "debug",
.mode = 0555,
.child = debug_table,
},
{
.procname = "dev",
.mode = 0555,
.child = dev_table,
},
/*
* NOTE: do not add new entries to this table unless you have read
* Documentation/sysctl/ctl_unnumbered.txt
*/
{ }
};
```


二、ioctl（网络相关部分）

一切均从系统调用开始，当用户调用ioctl函数时，会调用内核中的**SYSCALL_DEFINE3**函数，它是一个宏定义，如下：

` #define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)`


【注意】宏定义中的第一个#代表替换， 则代表使用’-’强制连接。
SYSCALL_DEFINEx之后调用__SYSCALL_DEFINEx函数，而__SYSCALL_DEFINEx同样是一个宏定义，如下：

```
#define __SYSCALL_DEFINEx(x, name, ...) \
asmlinkage long sys##name(__SC_DECL##x(__VA_ARGS__)); \
static inline long SYSC##name(__SC_DECL##x(__VA_ARGS__)); \
asmlinkage long SyS##name(__SC_LONG##x(__VA_ARGS__)) \
{ \
__SC_TEST##x(__VA_ARGS__); \
return (long) SYSC##name(__SC_CAST##x(__VA_ARGS__)); \
} \
SYSCALL_ALIAS(sys##name, SyS##name); \
static inline long SYSC##name(__SC_DECL##x(__VA_ARGS__))
```


中间会调用红色部分的宏定义，asmlinkage会通知编译器仅从**栈**中提取该函数的参数，所有的系统调用都会用到这个标志。

上面只是阐述了系统调用的一般过程，值得注意的是，上面这种系统调用过程的应用于最新的内核代码中，老版本中的这些过程是在syscall函数中完成的。我们就以在网络编程中ioctl系统调用为例介绍整个调用过程。当用户调用ioctl试图去从内核中获取某些值时，会触发调用：

```
SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
{
struct file *filp;
int error = -EBADF;
int fput_needed;
filp = fget_light(fd, &fput_needed); //根据进程描述符获取对应的文件对象
if (!filp)
goto out;
error = security_file_ioctl(filp, cmd, arg);
if (error)
goto out_fput;
error = do_vfs_ioctl(filp, fd, cmd, arg);
out_fput:
fput_light(filp, fput_needed);
out:
return error;
}
```


之后依次经过**file_ioctl---->vfs_ioctl**找到对应的与socket相对应的ioctl，即sock_ioctl.

```
static long vfs_ioctl(struct file *filp, unsigned int cmd,
unsigned long arg)
{
........
if (filp->f_op->unlocked_ioctl) {
error = filp->f_op->unlocked_ioctl(filp, cmd, arg);
if (error == -ENOIOCTLCMD)
error = -EINVAL;
goto out;
} else if (filp->f_op->ioctl) {
lock_kernel();
error = filp->f_op->ioctl(filp->f_path.dentry->d_inode,
filp, cmd, arg);
unlock_kernel();
}
.......
}
```


从上面代码片段中可以看出，根据对应的文件指针调用对应的ioctl，那么socket对应的文件指针的初始化是在哪完成的呢？可以参考socket.c文件下sock_alloc_file函数：

```
static int sock_alloc_file(struct socket *sock, struct file **f, int flags)
{
struct qstr name = { .name = "" };
struct path path;
struct file *file;
int fd;
..............
file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
&socket_file_ops);
if (unlikely(!file)) {
/* drop dentry, keep inode */
atomic_inc(&path.dentry->d_inode->i_count);
path_put(&path);
put_unused_fd(fd);
return -ENFILE;
}
.............
}
```


alloc_file函数将socket_file_ops指针赋值给socket中的f_op.同时注意**file->private_data = sock**这条语句。

```
struct file *alloc_file(struct path *path, fmode_t mode,const struct file_operations *fop)
{
.........
file->f_path = *path;
file->f_mapping = path->dentry->d_inode->i_mapping;
file->f_mode = mode;
file->f_op = fop;
file->private_data = sock;
........
}
```


而socket_file_ops是在socket.c文件中定义的一个静态结构体变量，它的定义如下：

```
static const struct file_operations socket_file_ops = {
.owner = THIS_MODULE,
.llseek = no_llseek,
.aio_read = sock_aio_read,
.aio_write = sock_aio_write,
.poll = sock_poll,
.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
.compat_ioctl = compat_sock_ioctl,
#endif
.mmap = sock_mmap,
.open = sock_no_open, /* special open code to disallow open via /proc */
.release = sock_close,
.fasync = sock_fasync,
.sendpage = sock_sendpage,
.splice_write = generic_splice_sendpage,
.splice_read = sock_splice_read,
};
```


从上面分析可以看出，**filp->f_op->unlocked_ioctl**实质调用的是sock_ioctl。

OK，再从sock_ioctl代码开始，如下：

```
static long sock_ioctl(struct file *file, unsigned cmd, unsigned long arg)
{
.......
sock = file->private_data;
sk = sock->sk;
net = sock_net(sk);
if (cmd >= SIOCDEVPRIVATE && cmd <= (SIOCDEVPRIVATE + 15)) {
err = dev_ioctl(net, cmd, argp);
} else
#ifdef CONFIG_WEXT_CORE
if (cmd >= SIOCIWFIRST && cmd <= SIOCIWLAST) {
err = dev_ioctl(net, cmd, argp);
} else
#endif
.......
default:
err = sock_do_ioctl(net, sock, cmd, arg);
break;
}
return err;
}
```


首先通过file变量的private_date成员将socket从sys_ioctl传递过来，最后通过执行sock_do_ioctl函数完成相应操作。

```
static long sock_do_ioctl(struct net *net, struct socket *sock,
unsigned int cmd, unsigned long arg)
{
........
err = sock->ops->ioctl(sock, cmd, arg);
.........
}
```


那么此时的socket中的ops成员又是从哪来的呢？我们以IPV4为例，都知道在创建socket时，都会需要设置相应的协议类型，此处的ops也是socket在创建inet_create函数中遍历inetsw列表得到的。

```
static int inet_create(struct net *net, struct socket *sock, int protocol,
int kern)
{
struct inet_protosw *answer;
........
sock->ops = answer->ops;
answer_prot = answer->prot;
answer_no_check = answer->no_check;
answer_flags = answer->flags;
rcu_read_unlock();
.........
}
```


那么inetsw列表又是在何处生成的呢？那是在协议初始化函数inet_init中调用inet_register_protosw（将全局结构体数组inetsw_array数组初始化）来实现的，

```
static struct inet_protosw inetsw_array[] =
{
{
.type = SOCK_STREAM,
.protocol = IPPROTO_TCP,
.prot = &tcp_prot,
.ops = &inet_stream_ops,
.no_check = 0,
.flags = INET_PROTOSW_PERMANENT |
INET_PROTOSW_ICSK,
},
{
.type = SOCK_DGRAM,
.protocol = IPPROTO_UDP,
.prot = &udp_prot,
.ops = &inet_dgram_ops,
.no_check = UDP_CSUM_DEFAULT,
.flags = INET_PROTOSW_PERMANENT,
},
{
.type = SOCK_RAW,
.protocol = IPPROTO_IP, /* wild card */
.prot = &raw_prot,
.ops = &inet_sockraw_ops,
.no_check = UDP_CSUM_DEFAULT,
.flags = INET_PROTOSW_REUSE,
}
};
```


很明显可以看到，**sock->ops->ioctl**需要根据具体的协议找到需要调用的ioctl函数。我们就以TCP协议为例，就需要调用**inet_stream_ops**中的ioctl函数——inet_ioctl，结构如下：

```
int inet_ioctl(struct socket *sock, unsigned int cmd, unsigned long arg)
{
struct sock *sk = sock->sk;
int err = 0;
struct net *net = sock_net(sk);
switch (cmd) {
case SIOCGSTAMP:
err = sock_get_timestamp(sk, (struct timeval __user *)arg);
break;
case SIOCGSTAMPNS:
err = sock_get_timestampns(sk, (struct timespec __user *)arg);
break;
case SIOCADDRT: //增加路由
case SIOCDELRT: //删除路由
case SIOCRTMSG:
err = ip_rt_ioctl(net, cmd, (void __user *)arg); //IP路由配置
break;
case SIOCDARP: //删除ARP项
case SIOCGARP: //获取ARP项
case SIOCSARP: //创建或者修改ARP项
err = arp_ioctl(net, cmd, (void __user *)arg); //ARP配置
break;
case SIOCGIFADDR: //获取接口地址
case SIOCSIFADDR: //设置接口地址
case SIOCGIFBRDADDR: //获取广播地址
case SIOCSIFBRDADDR: //设置广播地址
case SIOCGIFNETMASK: //获取网络掩码
case SIOCSIFNETMASK: //设置网络掩码
case SIOCGIFDSTADDR: //获取某个接口的点对点地址
case SIOCSIFDSTADDR: //设置每个接口的点对点地址
case SIOCSIFPFLAGS:
case SIOCGIFPFLAGS:
case SIOCSIFFLAGS: //设置接口标志
err = devinet_ioctl(net, cmd, (void __user *)arg); //网络接口配置相关
break;
default:
if (sk->sk_prot->ioctl)
err = sk->sk_prot->ioctl(sk, cmd, arg);
else
err = -ENOIOCTLCMD;
break;
}
return err;
}
```


到此，基本上找到了socket所对应的ioctl处理代码片段。整个流程大致可以用下面图进行概括（来自《深入理解Linux网络技术内幕》）：

三、Netlink

netlink已经成为用户空间与内核的IP网络配置之间的首选接口，同时它也可以作为内核内部与多个用户空间进程之间的消息传输系统.在实现netlink用于内核空间与用户空间之间的通信时，用户空间的创建方法和一般的套接字的创建使用类似，但内核的创建方法则有所不同，下图是netlink实现此类通信时的创建过程：

下面分别详细介绍内核空间与用户空间在实现此类通信时的创建方法：

**● 用户空间：**

创建流程大体如下：

① 创建socket套接字

② 调用bind函数完成地址的绑定，不过同通常意义下server端的绑定还是存在一定的差别的，server端通常绑定某个端口或者地址，而此处的绑定则是**将socket套接口与本进程的pid进行绑定**；

③ 通过sendto或者sendmsg函数发送消息；

④ 通过recvfrom或者rcvmsg函数接受消息。

【说明】

◆ netlink对应的协议簇是AF_NETLINK,协议类型可以是自定义的类型，也可以是内核预定义的类型；

```
#define NETLINK_ROUTE 0 /* Routing/device hook */
#define NETLINK_UNUSED 1 /* Unused number */
#define NETLINK_USERSOCK 2 /* Reserved for user mode socket protocols */
#define NETLINK_FIREWALL 3 /* Firewalling hook */
#define NETLINK_INET_DIAG 4 /* INET socket monitoring */
#define NETLINK_NFLOG 5 /* netfilter/iptables ULOG */
#define NETLINK_XFRM 6 /* ipsec */
#define NETLINK_SELINUX 7 /* SELinux event notifications */
#define NETLINK_ISCSI 8 /* Open-iSCSI */
#define NETLINK_AUDIT 9 /* auditing */
#define NETLINK_FIB_LOOKUP 10
#define NETLINK_CONNECTOR 11
#define NETLINK_NETFILTER 12 /* netfilter subsystem */
#define NETLINK_IP6_FW 13
#define NETLINK_DNRTMSG 14 /* DECnet routing messages */
#define NETLINK_KOBJECT_UEVENT 15 /* Kernel messages to userspace */
#define NETLINK_GENERIC 16
/* leave room for NETLINK_DM (DM Events) */
#define NETLINK_SCSITRANSPORT 18 /* SCSI Transports */
#define NETLINK_ECRYPTFS 19
```


上面是内核预定义的20种类型，当然也可以自定义一些。

◆ 前面说过，netlink处的绑定有着自己的特殊性，其需要绑定的协议地址可用以下结构来描述：

```
struct sockaddr_nl {
sa_family_t nl_family; /* AF_NETLINK */
unsigned short nl_pad; /* zero */
__u32 nl_pid; /* port ID */
__u32 nl_groups; /* multicast groups mask */
};
```


其中成员nl_family为AF_NETLINK，nl_pad当前未使用，需设置为0，成员nl_pid为接收或发送消息的进程的 ID，如果希望内核处理消息或多播消息，就把该字段设置为 0，否则设置为处理消息的进程 ID.，不过在此特别需要说明的是，此处是以进程为单位，倘若进程存在多个线程，那在与netlink通信的过程中如何准确找到对方线程呢？此时nl_pid可以这样表示：

`pthread_self() << 16 | getpid()`


pthread_self函数是用来获取线程ID，总之能够区分各自线程目的即可。成员** nl_groups用于指定多播组**，bind 函数用于把调用进程加入到该字段指定的多播组，**如果设置为 0，表示调用者不加入任何多播组**。

◆ 通过netlink发送的消息结构：

```
struct nlmsghdr {
__u32 nlmsg_len; /* Length of message including header */
__u16 nlmsg_type; /* Message type */
__u16 nlmsg_flags; /* Additional flags */
__u32 nlmsg_seq; /* Sequence number */
__u32 nlmsg_pid; /* Sending process port ID */
};
```


其中nlmsg_len指的是消息长度，nlmsg_type指的是消息类型，用户可以自己定义。字段nlmsg_flags 用于设置消息标志，对于一般的使用，用户把它设置为0 就可以，只是一些高级应用（如netfilter 和路由daemon 需要它进行一些复杂的操作），**字段 nlmsg_seq和 nlmsg_pid 用于应用追踪消息，前者表示顺序号，后者为消息来源进程 ID**。

**● 内核空间：**

内核空间主要完成以下三方面的工作：

① 创建netlinksocket，并注册回调函数，注册函数将会在有消息到达netlinksocket时会执行；

② 根据用户请求类型，处理用户空间的数据；

③ 将数据发送回用户。

【说明】

◆ netlink中利用netlink_kernel_create函数创建一个netlink socket.

`struct sock *netlink_kernel_create(struct net *net, int unit, unsigned int groups,void (*input)(struct sk_buff *skb),struct mutex *cb_mutex, struct module *module)`


net字段指的是网络的命名空间，一般用&init_net替代；**unit字段实质是netlink协议类型，值得注意的是，此值一定要与用户空间创建socket时的第三个参数值保持一致**；groups字段指的是socket的组名，一般置为0即可；input字段是回调函数，当netlink收到消息时会被触发；cb_mutex一般置为NULL；module一般置为THIS_MODULE宏。

◆ netlink是通过调用API函数netlink_unicast或者netlink_broadcast将数据返回给用户的.

` int netlink_unicast(struct sock *ssk, struct sk_buff *skb,u32 pid, int nonblock)`


ssk字段正是由netlink_kernel_create函数所返回的socket；参数skb指向的是socket缓存，它的**data字段用来指向要发送的netlink消息结构**;**参数pid为接收消息进程的pid，参数nonblock表示该函数是否为非阻塞，如果为1，该函数将在没有接收缓存可利用时立即返回，而如果为0，该函数在没有接收缓存可利用时睡眠**。

【引申】内核发送的netlink消息是通过struct sk_buffer结构来管理的，即socket缓存，linux/netlink.h中定义了

`#define NETLINK_CB(skb) (*(struct netlink_skb_parms*)&((skb)->cb))`


来方便消息的地址设置。

```
struct netlink_skb_parms {
struct ucred creds; /* Skb credentials */
__u32 pid;
__u32 dst_group;
kernel_cap_t eff_cap;
__u32 loginuid; /* Login (audit) uid */
__u32 sessionid; /* Session id (audit) */
__u32 sid; /* SELinux security id */
};
```


其中pid指的是发送者的进程ID，如：

`NETLINK_CB(skb).pid = 0; /*from kernel */`


◆ netlink API函数sock_release可以用来释放所创建的socket.



**【声明】转载时请注明出处和作者联系方式 文章出处：http://blog.csdn.net/chenlilong84/article/details/8235060 作者联系方式：chenlilong84@163.com**