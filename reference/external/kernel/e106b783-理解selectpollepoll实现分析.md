---
title: 理解select，poll，epoll实现分析
source: https://www.cnblogs.com/200911/p/7016843.html
kind: external
domain: kernel
author: 积淀
original_date: 2017-06-15
fetched_at: 2026-05-16
bookmark_title: 理解select，poll，epoll实现分析 - 积淀 - 博客园
tags: [external, kernel]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/200911/p/7016843.html)
> 作者：积淀
> 原始日期：2017-06-15
> 抓取日期：2026-05-16

# 理解select，poll，epoll实现分析

# 理解select，poll，epoll实现分析

mark 引用:http://janfan.cn/chinese/2015/01/05/select-poll-impl-inside-the-kernel.html 文章


# select()/poll() 的内核实现

05 Jan 2015同时对多个文件设备进行I/O事件监听的时候（I/O multiplexing），我们经常会用到系统调用函数`select()`

`poll()`

，甚至是为大规模成百上千个文件设备进行并发读写而设计的`epoll()`

。


I/O multiplexing: When an application needs to handle multiple I/O descriptors at the same time, and I/O on any one descriptor can result in blocking. E.g. file and socket descriptors, multiple socket descriptors

一旦某些文件设备准备好了，可以读写了，或者是我们自己设置的timeout时间到了，这些函数就会返回，根据返回结果主程序继续运行。

用了这些函数有什么好处？ 我们自己本来就可以实现这种I/O Multiplexing啊，比如说：

- 创建多个进程或线程来监听
- Non-blocking读写监听的轮询（polling）
- 异步I/O（Asynchronous I/O）与Unix Signal事件触发

想要和我们自己的实现手段做比较，那么首先我们就得知道这些函数在背后是怎么实现的。 本文以Linux（v3.9-rc8）源码为例，探索`select()`

`poll()`

的内核实现。

`select()`

源码概述

首先看看`select()`

函数的函数原型，具体用法请自行输入命令行`$ man 2 select`

查阅吧 : )

```
int select(int nfds,
fd_set *restrict readfds,
fd_set *restrict writefds,
fd_set *restrict errorfds,
struct timeval *restrict timeout);
```


下文将按照这个结构来讲解`select()`

在Linux的实现机制。

`select()`

内核入口`do_select()`

的循环体`struct file_operations`

设备驱动的操作函数`scull`

驱动实例`poll_wait`

与设备的等待队列- 其它相关细节
- 最后

好，让我们开始吧 : )

`select()`

内核入口

我们首先把目光放到文件`fs/select.c`

文件上。

```
SYSCALL_DEFINE5(select, int, n,
fd_set __user *, inp,
fd_set __user *, outp,
fd_set __user *, exp,
struct timeval __user *, tvp)
{
// …
ret = core_sys_select(n, inp, outp, exp, to);
ret = poll_select_copy_remaining(&end_time, tvp, 1, ret);
return ret;
}
```


```
int core_sys_select(int n,
fd_set __user *inp,
fd_set __user *outp,
fd_set __user *exp,
struct timespec *end_time)
{
fd_set_bits fds;
// …
if ((ret = get_fd_set(n, inp, fds.in)) ||
(ret = get_fd_set(n, outp, fds.out)) ||
(ret = get_fd_set(n, exp, fds.ex)))
goto out;
zero_fd_set(n, fds.res_in);
zero_fd_set(n, fds.res_out);
zero_fd_set(n, fds.res_ex);
// …
ret = do_select(n, &fds, end_time);
// …
}
```


很好，我们找到了一个宏定义的`select()`

函数的入口，继续深入，可以看到其中最重要的就是`do_select()`

这个内核函数。

`do_select()`

的循环体

`do_select()`

实质上是一个大的循环体，对每一个主程序要求监听的设备fd（File Descriptor）做一次`struct file_operations`

结构体里的`poll`

操作。

```
int do_select(int n, fd_set_bits *fds, struct timespec *end_time)
{
// …
for (;;) {
// …
for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
// …
struct fd f;
f = fdget(i);
if (f.file) {
const struct file_operations *f_op;
f_op = f.file->f_op;
mask = DEFAULT_POLLMASK;
if (f_op->poll) {
wait_key_set(wait, in, out,
bit, busy_flag);
// 对每个fd进行I/O事件检测
mask = (*f_op->poll)(f.file, wait);
}
fdput(f);
// …
}
}
// 退出循环体
if (retval || timed_out || signal_pending(current))
break;
// 进入休眠
if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
to, slack))
timed_out = 1;
}
}
```


`(*f_op->poll)`

会返回当前设备fd的状态（比如是否可读可写），根据这个状态，`do_select()`

接着做出不同的动作

- 如果设备fd的状态与主程序的感兴趣的I/O事件匹配，则记录下来，
`do_select()`

退出循环体，并把结果返回给上层主程序。 - 如果不匹配，
`do_select()`

发现timeout已经到了或者进程有signal信号打断，也会退出循环，只是返回空的结果给上层应用。

但如果`do_select()`

发现当前没有事件发生，又还没到timeout，更没signal打扰，内核会在这个循环体里面永远地轮询下去吗？

`select()`

把全部fd检测一轮之后如果没有可用I/O事件，会让当前进程去休眠一段时间，等待fd设备或定时器来唤醒自己，然后再继续循环体看看哪些fd可用，以此提高效率。

```
int poll_schedule_timeout(struct poll_wqueues *pwq, int state,
ktime_t *expires, unsigned long slack)
{
int rc = -EINTR;
// 休眠
set_current_state(state);
if (!pwq->triggered)
rc = schedule_hrtimeout_range(expires, slack, HRTIMER_MODE_ABS);
__set_current_state(TASK_RUNNING);
/*
* Prepare for the next iteration.
*
* The following set_mb() serves two purposes. First, it's
* the counterpart rmb of the wmb in pollwake() such that data
* written before wake up is always visible after wake up.
* Second, the full barrier guarantees that triggered clearing
* doesn't pass event check of the next iteration. Note that
* this problem doesn't exist for the first iteration as
* add_wait_queue() has full barrier semantics.
*/
set_mb(pwq->triggered, 0);
return rc;
}
EXPORT_SYMBOL(poll_schedule_timeout);
```


`struct file_operations`

设备驱动的操作函数

设备发现I/O事件时会唤醒主程序进程？ 每个设备fd的等待队列在哪？我们什么时候把当前进程添加到它们的等待队列里去了？

```
mask = (*f_op->poll)(f.file, wait);
```


就是上面这行代码干的好事。 不过在此之前，我们得先了解一下系统内核与文件设备的驱动程序之间耦合框架的设计。

上文对每个设备的操作`f_op->poll`

，是一个针对每个文件设备特定的内核函数，区别于我们平时用的系统调用`poll()`

。 并且，这个操作是`select()`

`poll()`

`epoll()`

背后实现的共同基础。

Support for any of these calls requires support from the device driver. This support (for all three calls,

`select()`

`poll()`

and`epoll()`

) is provided through the driver’s poll method.

Linux的设计很灵活，它并不知道每个具体的文件设备是怎么操作的（怎么打开，怎么读写），但内核让每个设备拥有一个`struct file_operations`

结构体，这个结构体里定义了各种用于操作设备的函数指针，指向操作每个文件设备的驱动程序实现的具体操作函数，即设备驱动的回调函数（callback）。

```
struct file {
struct path f_path;
struct inode *f_inode; /* cached value */
const struct file_operations *f_op;
// …
} __attribute__((aligned(4))); /* lest something weird decides that 2 is OK */
```


```
struct file_operations {
struct module *owner;
loff_t (*llseek) (struct file *, loff_t, int);
ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
int (*iterate) (struct file *, struct dir_context *);
// select()轮询设备fd的操作函数
unsigned int (*poll) (struct file *, struct poll_table_struct *);
// …
};
```


这个`f_op->poll`

对文件设备做了什么事情呢？ 一是调用`poll_wait()`

函数（在`include/linux/poll.h`

文件）； 二是检测文件设备的当前状态。

```
unsigned int (*poll) (struct file *filp, struct poll_table_struct *pwait);
```


The device method is in charge of these two steps:


- Call
`poll_wait()`

on one or more wait queues that could indicate a change in the poll status. If no file descriptors are currently available for I/O, the kernel causes the process to wait on the wait queues for all file descriptors passed to the system call.- Return a bit mask describing the operations (if any) that could be immediately performed without blocking.

或者来看另一个版本的说法：

For every file descriptor, it calls that fd’s

`poll()`

method, which will add the caller to that fd’s wait queue, and return which events (readable, writeable, exception) currently apply to that fd.

下一节里我们会结合驱动实例程序来理解。

`scull`

驱动实例

由于Linux设备驱动的耦合设计，对设备的操作函数都是驱动程序自定义的，我们必须要结合一个具体的实例来看看，才能知道`f_op->poll`

里面弄得是什么鬼。

在这里我们以Linux Device Drivers, Third Edition一书中的例子——`scull`

设备的驱动程序为例。


`scull`

(Simple Character Utility for Loading Localities). scull is a char driver that acts on a memory area as though it were a device.

`scull`

设备不同于硬件设备，它是模拟出来的一块内存，因此对它的读写更快速更自由，内存支持你顺着读倒着读点着读怎么读都可以。 我们以书中“管道”（pipe）式，即FIFO的读写驱动程序为例。

首先是`scull_pipe`

的结构体，注意`wait_queue_head_t`

这个队列类型，它就是用来记录等待设备I/O事件的进程的。

```
struct scull_pipe {
wait_queue_head_t inq, outq; /* read and write queues */
char *buffer, *end; /* begin of buf, end of buf */
int buffersize; /* used in pointer arithmetic */
char *rp, *wp; /* where to read, where to write */
int nreaders, nwriters; /* number of openings for r/w */
struct fasync_struct *async_queue; /* asynchronous readers */
struct mutex mutex; /* mutual exclusion semaphore */
struct cdev cdev; /* Char device structure */
};
```


`scull`

设备的轮询操作函数`scull_p_poll`

，驱动模块加载后，这个函数就被挂到`(*poll)`

函数指针上去了。

我们可以看到它的确是返回了当前设备的I/O状态，并且调用了内核的`poll_wait()`

函数，这里注意，它把自己的`wait_queue_head_t`

队列也当作参数传进去了。

```
static unsigned int scull_p_poll(struct file *filp, poll_table *wait)
{
struct scull_pipe *dev = filp->private_data;
unsigned int mask = 0;
/*
* The buffer is circular; it is considered full
* if "wp" is right behind "rp" and empty if the
* two are equal.
*/
mutex_lock(&dev->mutex);
poll_wait(filp, &dev->inq, wait);
poll_wait(filp, &dev->outq, wait);
if (dev->rp != dev->wp)
mask |= POLLIN | POLLRDNORM; /* readable */
if (spacefree(dev))
mask |= POLLOUT | POLLWRNORM; /* writable */
mutex_unlock(&dev->mutex);
return mask;
}
```


当`scull`

有数据写入时，它会把`wait_queue_head_t`

队列里等待的进程给唤醒。

```
static ssize_t scull_p_write(struct file *filp, const char __user *buf, size_t count,
loff_t *f_pos)
{
// …
/* Make sure there's space to write */
// …
/* ok, space is there, accept something */
// …
/* finally, awake any reader */
wake_up_interruptible(&dev->inq); /* blocked in read() and select() */
// …
}
```


可是`wait_queue_head_t`

队列里的进程是什么时候装进去的？ 肯定是`poll_wait`

搞的鬼！ 我们又得回到该死的Linux内核去了。

`poll_wait`

与设备的等待队列

```
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
if (p && p->_qproc && wait_address)
p->_qproc(filp, wait_address, p);
}
/*
* Do not touch the structure directly, use the access functions
* poll_does_not_wait() and poll_requested_events() instead.
*/
typedef struct poll_table_struct {
poll_queue_proc _qproc;
unsigned long _key;
} poll_table;
/*
* structures and helpers for f_op->poll implementations
*/
typedef void (*poll_queue_proc)(struct file *, wait_queue_head_t *, struct poll_table_struct *);
```


可以看到，`poll_wait()`

其实就是只是直接调用了`struct poll_table_struct`

结构里绑定的函数指针。 我们得找到`struct poll_table_struct`

初始化的地方。

The

`poll_table`

structure is just a wrapper around a function that builds the actual data structure. That structure, for`poll`

and`select`

, is a linked list of memory pages containing`poll_table_entry`

structures.

`struct poll_table_struct`

里的函数指针，是在`do_select()`

初始化的。

```
int do_select(int n, fd_set_bits *fds, struct timespec *end_time)
{
struct poll_wqueues table;
poll_table *wait;
poll_initwait(&table);
wait = &table.pt;
// …
}
void poll_initwait(struct poll_wqueues *pwq)
{
// 初始化poll_table里的函数指针
init_poll_funcptr(&pwq->pt, __pollwait);
pwq->polling_task = current;
pwq->triggered = 0;
pwq->error = 0;
pwq->table = NULL;
pwq->inline_index = 0;
}
EXPORT_SYMBOL(poll_initwait);
static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
pt->_qproc = qproc;
pt->_key = ~0UL; /* all events enabled */
}
```


我们现在终于知道，`__pollwait()`

函数，就是`poll_wait()`

幕后的真凶。

`add_wait_queue()`

把当前进程添加到设备的等待队列`wait_queue_head_t`

中去。

```
/* Add a new entry */
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
poll_table *p)
{
struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
struct poll_table_entry *entry = poll_get_entry(pwq);
if (!entry)
return;
entry->filp = get_file(filp);
entry->wait_address = wait_address;
entry->key = p->_key;
init_waitqueue_func_entry(&entry->wait, pollwake);
entry->wait.private = pwq;
// 把当前进程装到设备的等待队列
add_wait_queue(wait_address, &entry->wait);
}
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
unsigned long flags;
wait->flags &= ~WQ_FLAG_EXCLUSIVE;
spin_lock_irqsave(&q->lock, flags);
__add_wait_queue(q, wait);
spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(add_wait_queue);
static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
list_add(&new->task_list, &head->task_list);
}
/**
* Insert a new element after the given list head. The new element does not
* need to be initialised as empty list.
* The list changes from:
* head → some element → ...
* to
* head → new element → older element → ...
*
* Example:
* struct foo *newfoo = malloc(...);
* list_add(&newfoo->entry, &bar->list_of_foos);
*
* @param entry The new element to prepend to the list.
* @param head The existing list.
*/
static inline void
list_add(struct list_head *entry, struct list_head *head)
{
__list_add(entry, head, head->next);
}
```


## 其它相关细节

`fd_set`

实质上是一个`unsigned long`

数组，里面的每一个`long`

整值的每一位都代表一个文件，其中置为1的位表示用户要求监听的文件。 可以看到，`select()`

能同时监听的fd好少，只有1024个。

```
#define __FD_SETSIZE 1024
typedef struct {
unsigned long fds_bits[__FD_SETSIZE / (8 * sizeof(long))];
} __kernel_fd_set;
typedef __kernel_fd_set fd_set;
```


- 所谓的文件描述符fd (File Descriptor)，大家也知道它其实只是一个表意的整数值，更深入地说，它是每个进程的file数组的下标。

```
struct fd {
struct file *file;
unsigned int flags;
};
```


`select()`

系统调用会创建一个`poll_wqueues`

结构体，用来记录相关I/O设备的等待队列；当`select()`

退出循环体返回时，它要把当前进程从全部等待队列中移除——这些设备再也不用着去唤醒当前队列了。

The call to

`poll_wait`

sometimes also adds the process to the given wait queue. The whole structure must be maintained by the kernel so that the process can be removed from all of those queues before poll or select returns.

```
/*
* Structures and helpers for select/poll syscall
*/
struct poll_wqueues {
poll_table pt;
struct poll_table_page *table;
struct task_struct *polling_task;
int triggered;
int error;
int inline_index;
struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};
struct poll_table_entry {
struct file *filp;
unsigned long key;
wait_queue_t wait;
wait_queue_head_t *wait_address;
};
```


`wait_queue_head_t`

就是一个进程（task）的队列。

```
struct __wait_queue_head {
spinlock_t lock;
struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```


`select()`

与`epoll()`

的比较


- select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用
`epoll_wait`

不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。- epoll所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以
`cat /proc/sys/fs/file-max`

察看,一般来说这个数目和系统内存关系很大。

更具体的比较可以参见这篇文章。

## 最后

非常艰难的，我们终于来到了这里（T^T）

总结一下`select()`

的大概流程（`poll`

同理，只是用于存放fd的数据结构不同而已）。

- 先把全部fd扫一遍
- 如果发现有可用的fd，跳到5
- 如果没有，当前进程去睡觉xx秒
- xx秒后自己醒了，或者状态变化的fd唤醒了自己，跳到1
- 结束循环体，返回

我相信，你肯定还没懂，这代码实在是乱得一逼，被我剪辑之后再是乱得没法看了（叹气）。 所以看官请务必亲自去看Linux源码，在这里我已经给出了大致的方向，等你看完源码回来，这篇文章你肯定也就明白了。 当然别忘了下面的参考资料，它们可帮大忙了 :P