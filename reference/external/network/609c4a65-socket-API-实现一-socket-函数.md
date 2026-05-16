---
title: socket API 实现（一）—— socket 函数
source: http://blog.guorongfei.com/2014/10/23/socket-create/
kind: external
domain: network
original_date: 2014-10-23
fetched_at: 2026-05-16
bookmark_title: socket API 实现（一）—— socket 函数 | 时习之
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.guorongfei.com](http://blog.guorongfei.com/2014/10/23/socket-create/)
> 原始日期：2014-10-23
> 抓取日期：2026-05-16

# socket API 实现（一）—— socket 函数

**注：文中使用到内核版本是 2.6.32.63，其他版本可能会有些许出入**

# 概述

如果我们在应用程序中创建一个 socket，我们一般使用类似下面这样的语句：

1 | tcp_scokt = socket(AF_INET, SOCK_STREAM, 0); |

那么这条语句在内核的内部到底做了什么呢？这一类到文章和书籍已经非常的多，讲解的也 非常的透彻，所以这篇文章并不想讲解过多的代码，如果你需要了解具体的代码，可以查看 我给出的参考链接[^1][^2]。

参考链接里面的资料为求透彻，基本上都是跟踪代码的运行，通过一层层的函数调用关系来 解析 socket 创建到整个流程。这种做法在虽然在深度上能够让读者有抽丝剥茧的感觉，但 是最终因为代码的复杂性使得读者容易迷失在代码之中。

这篇文章以总结、概括到方式，从数据结构之间的关系和重要的全局变量入手，给大家搭建 一个基本的框架。你可以在阅读完具体的代码之后回过头来看这篇文章，从整体上有个把握 。也可以在阅读代码之前先看看这篇文章从而有一个总体的把握。

# 图示

首先给一张简单到草图，概括 socket 创建到整个流程，这里只列出关键的函数：

**在上图中标有 * 号的函数表示不是直接调用，而是通过函数指针间接调用的。**

# struct socket、struct sock、struct file 的关系

在 **上一篇** 文章中我们提到，socket 它在层次结构上来说是位于
用户程序和内核协议栈之间的一个接口，而在实现上它是一种特殊的文件。

socket 的实现其实非常的复杂，对于不同的层次抽象出了不同的接口和数据结构。对于
socket layer 有 struct socket 这个结构，而对于内核具体的实现则存在 struct sock
这个结构。这两个结构可以说是一个概念的两面，struct socket 结构体很简单，因为它是
开放出来的接口^3，越简单越好；而 struct sock 这个接口则用于具体的协议栈内部内
部实现。它们两个是一一对应的关系，互相引用。在上图中的 `sock_init_data`

中存在下
面两行代码：

1 | sk_set_socket(sk, sock); |

其中第一句最终执行的代码是 `sk->sk_socket = sock;`

用来使 sock 结构体指向 socket
结构体而第二句的作用是使得 socket 结构体指向 sock结构体。如此一来，它们两者就一
一对应了。

不过 socket 结构不是给用户的接口，对于用户来说 socket 是文件，访问 socket 是通过
文件描述符来完成的。而面向文件系统，表达文件的概念不是 socket 结构而是 struct
file 结构。对于 socket 来说，这两个结构也是一一对应的，在上图的 `sock_attach_fd`

函数中有下面两条语句：

1 | sock->file = file; |

第一条语句让 socket 结构体得到对于的 file 结构体的引用，而第二条语句则让 file 结 构保存对 socket 结构体的引用。所以最后的结果是任意给出其中的一者都可以访问到其他 两者。

不过对于用户空间的程序来说， 它们看到的 socket 是一个文件描述符而不是 struct file 结构，这中间还有一层转换。那就是在每个进程的进程描述符中都可以找到文件描述 符 fd 和 struct file 结构体之间的映射表，用不是非常精确的图来表示的话相当于下图 ：

当用户使用文件描述符 fd 来访问 socket 的时候，首先通过查找 fd 到 file 结构的映射 表得到 file 结构，然后通过 file 结构的 private_data 得到 socket 结构，进而可以得 到 sock 结构。在后续的 socket API（比如：bind()，listen()）中都是通过这种方式得 到 sock 结构体的。

# sock,inet_sock,inet_connection_sock,tcp_sock

在 socket 函数关系调用图中，非常重要的一步是调用 `sk_alloc`

为 `struct socket`

结构分配 `struct sock`

结构空间。但是 `sk_alloc`

函数分配的其实并不是 ```
struct
scok
```

结构空间而是分配了 `strcut tcp_sock`

结构的空间。下面这段是具体的代码：

1 | slab = prot->slab; |

从上面这段代码我们可以看出，如果 slab 存在，则用专用的缓存来分配 sock，如果不存
在的话，则在通用缓存中分配 prot->obj_size 大小的空间作为 sock 的空间。对于 tcp
socket 来说。 `prot`

变量是 `tcp_prot`

，从它的定义中我们可以看到它的 `obj_size`

字段是 `tcp_sock`

的大小，初始化代码如下：

1 | .obj_size = sizeof(struct tcp_sock), |

不过，`tcp_prot`

本身是存在 slab 的，在 `inet_init`

函数中存在下面这条函数调用：

1 | rc = proto_register(&tcp_prot, 1); |

在 `proto_register`

中则有如下代码：

1 | prot->slab = kmem_cache_create(prot->name, prot->obj_size, 0, |

这段代码用于分配 `tcp_prot`

结构的 slab 字段缓存，而这个缓存的元素大小还是
`obj_size`

。所以在 `sk_alloc`

中最后返回的空间大小为 `tcp_sock`

的大小，而不是
`sock`

的大小。

但是代码中把 `tcp_sock`

转换成了 `sock`

和 `inet_sock`

指针来进行初始化。之后又
在 `tcp_v4_init_sock`

中把他转换成 `inet_connection_sock`

和 `tcp_sock`

指针进行
初始化。

之所以可以这么做，是因为这几个结构中存在一种包含关系，下面是这几个结构的定义：

1 | struct sock { |

可以看到，后面的每一个结构的定义都把前面的结构包含在结构的最开头的位置。这样一来 ，他们直接就形成了一种层次话的关系。这是一种变相的继承方式，可以把一个子类“向上 转型”为父类。个人认为这种方式是 c 语言中实现继承和多态的手法。

在 `sk_alloc`

中分配一个 `tcp_sock`

空间之后转换成 `sock`

，`inet_sock`

，
`inet_connetion_sock`

相当于下图：

所以他们之间向上一级进行转换是不会有问题的。

# 重点的静态全局变量

sock_mnt

这个变量我们在

**上一篇**中有提到。在初始化好了 socket 文 件系统之后，最终的挂载点结构`vfsmount`

存放在`sock_mnt`

这个结构体中。`sys_socket`

系统调用的第一步就是分配一个 socket 结构空间，而这个结构就是通过`sock_mnt`

来创建的。在`sock_mnt`

结构中存放了 socket 文件系统的`super block`

，而在`super block`

中有分配 inode 的函数。对于 socket 文件系统来说， 这个分配函数分配的是`socket_alloc`

结构。`sock_alloc`

返回的就是这个结构体 的 socket 成员。具体的请查看**上一篇**文章。net_families

我们给 socket() 函数传递的三个参数其实是层层递进的关系。首先用 domain 指明协 议簇，然后通过第二个参数 type 指明该协议簇下面的协议类型，最后通过第三个参数 protocol 指明这种类型的具体协议。

不同的协议簇在创建 socket 结构的时候使用的方法不一样，为了使得接口统一，抽象 了

`net_proto_family`

这个结构。它里面有一个指针函数成员 create 用来设置前面 通过`sock_mnt`

分配的 socket 结构。像前面的 fd 和 file 结构的关系一样，

`socket()`

函数提供的 AF_INET 只是一个标 志而已，而不是真正的`net_proto_family`

结构。要找到真正的`net_proto_family`

则需要通过另一个映射表。这个映射表就是`net_families`

。它是定义在 net/socktet.c 中的全局变量：

1 | static const struct net_proto_family *net_families[NPROTO] __read_mostly; |

在代码中通过:

1 | pf = rcu_dereference(net_families[family]); |

这行代码，对 `net_families`

进行下标索引找到我们想要的 ```
struct
net_proto_family
```

结构。而 `net_families`

中的内容则是在各个协议簇初始化的时
候通过 `sock_register`

注册进去的。

inetsw

这个全局变量和前面的

`net_families`

功能可以说是一样的，也是用来做映射，不同 的是`inetsw`

的映射是用于第二个参数和第三个参数的。`inetsw`

的 “sw” 后缀就是 “switch” 的缩写，从命名上就可以看出它是用来做映射转 换的。`inetsw`

是一个数组，每一个 socket type 都会在它里面有一个对应的元素。 不过和`net_families`

不同的是，`inetsw`

中的元素不是唯一确定的，而是一个链表 ，而这个链表就是用于第三个参数的转换了。在每一个

`inetsw`

的链表中，而每一个链表成员都是`struct inet_protosw`

结构体 。最终通过比较这个结构体的 protocol 字段和传入进来的 protocol 参数来找到最终 的`struct inet_protosw`

。如果我们使用`socket(AF_INET, SOCK_STREAM, 0);`

调 用，经过前面这几次转换之后最终得到的`struct inet_protosw`

结构变量是一个描述 tcp 协议的结构体。`struct inet_protosw`

也有 “sw” 后缀，因为它本身也相当于一个转换器，不过前面 提到到转换不同的是这里的转换是在`socket layer`

和具体实现之间的转换，这个结 构体中有如下两个成员：

1 | struct proto *prot; |

这两个结构体中定义了大量的同名的函数指针，只不过参数不一样。比如 `listen`

这
个函数，在 struct proto_ops 中的定义是：

1 | int (*bind)(struct socket *sock, |

而在 struct proto 中的定义是这样:

1 | int (*bind)(struct sock *sk, |

两个函数函数原型除除了在第一个参数不同之外其他都是一样的。而正是从这个参数的
区别中我们可以看出它们两者在应用范围上的不同。前者使用 `struct socket`

作为参
数而后者使用 `struct sock`

作为参数，这说明前者主要是面向 `socket layer`

的接
口，而后者则是内核协议栈具体实现中的接口。我想这大概也是为什么 ```
struct
inet_protosw
```

为何以 “sw” 结尾的原因。

tcp_prot, inet_stream_ops

前面提到

`socket inet_protosw`

面向不同的层次提供了两个不同的结构，而对于 tcp 协议来说这两个接口分别是`tcp_proto`

（面向实现内部）和`inet_stream_ops`

（ 面向 socket layer）。在 struct socket 结构体的定义中有如下字段：

1 | const struct proto_ops *ops; |

在 `inet_create`

它被赋值成了 `inet_stream_ops`

（当然这是对于创建 tcp socket
来说）。而在 struct sock 中定义了如下两个字段：

1 |
|

也就是说 sock 结构体中包含了 proto 相关的字段，这两个字段在 sock 的初始化的时
候都被初始化成了 `tcp_prot`

。

前面的 `sk_prot`

字段其实只是宏定义，因为真正对应的的 `struct proto`

是定义在
`sk_common`

中的字段。而 sock 结构体定义了一个类型是 `sk_common`

的字段
`__sk_common`

。

# 总结

用户通过得到的文件描述符 `sock_fd`

可以找到对于的 `struct file`

结构，而 ```
struct
file
```

结构有 `struct socket`

的引用，后者则包含了 `struct sock`

的引用。 在
`struct socket`

中有一套面向 `socket layer`

中的接口 `struct proto_ops`

（
`inet_stream_ops`

） 而在 `struct sock`

中有具体的协议的接口 `struct proto`

（
`tcp_prot`

）。

[^2]: 《追踪 Linux TCP/IP 代码运行——基于 2.6 内核》—— 秦健 编著