---
title: 使用Valgrind和ThreadSanitizer检测多线程错误
source: https://segmentfault.com/a/1190000005102690
kind: external
domain: compute
author: Spacewander
original_date: 2016-05-11
fetched_at: 2026-05-16
bookmark_title: 使用Valgrind和ThreadSanitizer检测多线程错误 - spacewander - SegmentFault 思否
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[segmentfault.com](https://segmentfault.com/a/1190000005102690)
> 作者：Spacewander
> 原始日期：2016-05-11
> 抓取日期：2026-05-16

# 使用Valgrind和ThreadSanitizer检测多线程错误

做毕设的时候，我曾经遇到一个多线程的BUG。这个BUG表现得较为诡异，会导致数据随机出错。由于找不出什么规律，一开始我还是挺头疼的。查了半天后我发现，相关的日志有多线程下共享数据访问问题的迹象（即所谓的data race），所以很快确诊是多线程部分代码存在逻辑错误。这个问题的解决办法很简单，就是把相关的代码review下，找出data race的部分并加以修正。虽然BUG是搞定了，不过我还是想找到一个自动化工具，能够检测出代码中潜在的线程安全问题。这样就能把BUG消灭在萌芽之中，而不是等到事后才睁大眼睛揪它出来。

搜索了下，发现了两个适合做这个的工具，Valgrind和ThreadSanitizer。今天就来介绍下这两个工具。

## Valgrind

Valgrind一般用做内存泄露和访存越界检测，除此之外，其实它也支持对data race及一些简单的多线程问题的检查。Valgrind工具集里面，helgrind和drd都能用来完成这种检测。你可以用`valgrind --tool=helgrind`

或`valgrind --tool=drd`

来启用它。只要应用使用的线程模型是POSIX thread（pthread），这两个工具就能进行检测。这两个工具间差别不大，下面我就基于`helgrind`

来介绍下用法：

先上一段有问题的示例代码：

```
// raceCondition.cpp
#include <pthread.h>
void *write_buffer(void *args)
{
pthread_t *buffer = static_cast<pthread_t *>(args);
*buffer = pthread_self();
pthread_exit(0);
return NULL;
}
int main()
{
pthread_t *buffer = new pthread_t[2];
pthread_t a, b;
pthread_create(&a, NULL, write_buffer, buffer);
pthread_create(&b, NULL, write_buffer, buffer);
pthread_join(a, NULL);
pthread_join(b, NULL);
delete []buffer;
return 0;
}
```


这段代码有一个刻意为之的问题，线程a和线程b写入了同一个缓冲区。

用Valgrind可以检测出问题：

```
==5697== ---Thread-Announcement------------------------------------------
==5697==
==5697== Thread #3 was created
==5697== at 0x545943E: clone (clone.S:74)
==5697== by 0x5148199: do_clone.constprop.3 (createthread.c:75)
==5697== by 0x51498BA: create_thread (createthread.c:245)
==5697== by 0x51498BA: pthread_create@@GLIBC_2.2.5 (pthread_create.c:611)
==5697== by 0x4C30E0D: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==5697== by 0x400928: main (in /home/lzx/C/thread_error/a.out)
==5697==
==5697== ---Thread-Announcement------------------------------------------
==5697==
==5697== Thread #2 was created
==5697== at 0x545943E: clone (clone.S:74)
==5697== by 0x5148199: do_clone.constprop.3 (createthread.c:75)
==5697== by 0x51498BA: create_thread (createthread.c:245)
==5697== by 0x51498BA: pthread_create@@GLIBC_2.2.5 (pthread_create.c:611)
==5697== by 0x4C30E0D: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==5697== by 0x40090B: main (in /home/lzx/C/thread_error/a.out)
==5697==
==5697== ---Thread-Announcement------------------------------------------
==5697==
==5697== Thread #1 is the program's root thread
==5697==
==5697== ----------------------------------------------------------------
==5697==
==5697== Possible data race during write of size 8 at 0x5C40040 by thread #3
==5697== Locks held: none
==5697== at 0x4008BB: write_buffer(void*) (in /home/lzx/C/thread_error/a.out)
==5697== by 0x4C30FA6: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==5697== by 0x5149181: start_thread (pthread_create.c:312)
==5697== by 0x545947C: clone (clone.S:111)
==5697==
==5697== This conflicts with a previous write of size 8 by thread #2
==5697== Locks held: none
==5697== at 0x4008BB: write_buffer(void*) (in /home/lzx/C/thread_error/a.out)
==5697== by 0x4C30FA6: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==5697== by 0x5149181: start_thread (pthread_create.c:312)
==5697== by 0x545947C: clone (clone.S:111)
==5697== Address 0x5c40040 is 0 bytes inside a block of size 16 alloc'd
==5697== at 0x4C2CC20: operator new[](unsigned long) (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==5697== by 0x4008EA: main (in /home/lzx/C/thread_error/a.out)
==5697== Block was alloc'd by thread #1
```


输出结果包括data race的内存位置、内存区域大小和涉及的线程，以及调用栈。

如果编译程序时加了`-g`

选项，那么输出的调用栈中会有具体的位置：

```
==7993== This conflicts with a previous write of size 8 by thread #2
==7993== Locks held: none
==7993== at 0x4008BB: write_buffer(void*) (raceCondition.cpp:8)
==7993== by 0x4C30FA6: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7993== by 0x5149181: start_thread (pthread_create.c:312)
==7993== by 0x545947C: clone (clone.S:111)
==7993== Address 0x5c40040 is 0 bytes inside a block of size 16 alloc'd
==7993== at 0x4C2CC20: operator new[](unsigned long) (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7993== by 0x4008EA: main (raceCondition.cpp:17)
==7993== Block was alloc'd by thread #1
```


Valgrind记录了每个线程的内存访问情况，如果多个线程对同一个内存地址的访问没有限定次序（诸如happen before这样的memory model细则），就会被判为“Possible data race”。

Valgrind的检测同样对C++11提供的thread库生效（只要底层用的还是pthread）：

```
#include <thread>
using namespace std;
void write_buffer(thread::id *buffer)
{
*buffer = this_thread::get_id();
}
int main()
{
thread::id *buffer = new thread::id[2];
thread a(write_buffer, buffer);
thread b(write_buffer, buffer);
a.join();
b.join();
return 0;
}
```


输出报告跟pthread版本的差不多。由于输出太长，这里就不贴了。

除了data race，Valgrind也能检测出一些简单的多线程问题，比如线程结束时没有释放锁：

```
#include <pthread.h>
pthread_mutex_t mutex;
void *still_locked(void *args)
{
(void)args;
pthread_mutex_lock(&mutex);
pthread_exit(0);
return NULL;
}
int main()
{
pthread_mutex_init(&mutex, NULL);
pthread_t a;
pthread_create(&a, NULL, still_locked, NULL);
pthread_join(a, NULL);
return 0;
}
```


```
==6316== Thread #2: Exiting thread still holds 1 lock
==6316== at 0x4E4521F: start_thread (pthread_create.c:457)
```


即使线程detach了也能检测出来。

```
#include <pthread.h>
pthread_mutex_t mutex;
void *still_locked(void *args)
{
(void)args;
pthread_detach(pthread_self());
pthread_mutex_lock(&mutex);
pthread_exit(0);
return NULL;
}
int main()
{
pthread_mutex_init(&mutex, NULL);
pthread_t a;
pthread_create(&a, NULL, still_locked, NULL);
return 0;
}
```


```
==6574== Thread #2: Exiting thread still holds 1 lock
==6574== at 0x4E4521F: start_thread (pthread_create.c:457)
```


## ThreadSanitizer

ThreadSanitizer是另外一个检测多线程问题的工具，集成于gcc 4.8和clang 3.2以上的版本。

换句话说，只要你的编译器版本不太旧，那么你就可以立刻启用它。

对于clang，需要使用下列的编译/链接选项：

`clang -fsanitize=thread -fPIE -pie -g`


对于gcc，可能还要加上-ltsan

`gcc -fsanitize=thread -fPIE -pie -g -ltsan`


如果出现了链接错误，检查下是否有libtsan这个库。

以上节展示的第一段代码为例：

```
$ g++ raceCondition.cpp -fsanitize=thread -fPIE -pie -g -ltsan
$ ./a.out
==================
WARNING: ThreadSanitizer: data race (pid=8425)
Write of size 8 at 0x7d020000eff0 by thread T2:
#0 write_buffer(void*) /home/lzx/C/thread_error/raceCondition.cpp:8 (exe+0x000000000c1b)
#1 __tsan_write_range ??:0 (libtsan.so.0+0x00000001b1c9)
Previous write of size 8 at 0x7d020000eff0 by thread T1:
#0 write_buffer(void*) /home/lzx/C/thread_error/raceCondition.cpp:8 (exe+0x000000000c1b)
#1 __tsan_write_range ??:0 (libtsan.so.0+0x00000001b1c9)
Location is heap block of size 16 at 0x7d020000eff0 allocated by main thread:
#0 operator new[](unsigned long) ??:0 (libtsan.so.0+0x00000001cfe2)
#1 main /home/lzx/C/thread_error/raceCondition.cpp:17 (exe+0x000000000c5b)
Thread T2 (tid=8427, running) created by main thread at:
#0 pthread_create ??:0 (libtsan.so.0+0x00000001eccb)
#1 main /home/lzx/C/thread_error/raceCondition.cpp:21 (exe+0x000000000c9d)
Thread T1 (tid=8426, finished) created by main thread at:
#0 pthread_create ??:0 (libtsan.so.0+0x00000001eccb)
#1 main /home/lzx/C/thread_error/raceCondition.cpp:20 (exe+0x000000000c7e)
SUMMARY: ThreadSanitizer: data race /home/lzx/C/thread_error/raceCondition.cpp:8 write_buffer(void*)
==================
ThreadSanitizer: reported 1 warnings
```


输出结果跟Valgrind的大同小异。ThreadSanitizer的检测机制跟Valgrind相似，也是检测各线程对内存的访问是否有序。不同的是，ThreadSanitizer会在编译时给特定的访存操作注入监控指令，而不是在运行时监控全部的访存操作。这么一来，ThreadSanitizer的内存占用和性能损耗会比Valgrind的少很多，这也是它的主打优点。

ThreadSanitizer的另一个主打优点是，它支持的data race检测要比Valgrind的更多。

不过，在某些方面（比如上文提到的线程结束时没有释放锁）的检测，ThreadSanitizer却又不如Valgrind。

## 结论

这两个工具之间，我偏好Valgrind。在资源占用方面，除非你的项目已经达到Chrome级别，否则不用太在意运行测试的用时；在功能方面，两者间差异不大；而ThreadSanitizer用起来相对麻烦一些。它需要特定的编译指令，一旦跟现有的编译方式冲突就很蛋疼了。

事实上，如果要我给这两个工具打分，满分100我只能给70.

这两个工具的输出都很含糊。Possible data race？Previous write by thread X？在现实应用中使用时，出问题之处要比上述的示例代码难理解多了。而且，Valgrind的多线程问题检测有一定可能出现误报。（之前在毕设的应用中就遇到过）

另外，只有进行了内存访问才会触发data race的检测。对于一类小概率触发的data race问题，这两个工具不一定能检测出来。

写一个randomRaceCondition.cpp作为例子：

```
#include <cstdlib>
#include <ctime>
#include <pthread.h>
void *write_buffer(void *args) {
pthread_t *buffer = static_cast<pthread_t *>(args);
if (rand() % 2 == 0) { // 现在线程a和b都进行访存操作的概率为1/4
*buffer = pthread_self();
}
pthread_exit(0);
return NULL;
}
int main()
{
srand(time(0));
pthread_t *buffer = new pthread_t[2];
pthread_t a, b;
pthread_create(&a, NULL, write_buffer, buffer);
pthread_create(&b, NULL, write_buffer, buffer);
pthread_join(a, NULL);
pthread_join(b, NULL);
delete []buffer;
return 0;
}
```


无论是Valgrind还是ThreadSanitizer，在单次运行内检测出data race都是个随机事件了。

最后，Valgrind/ThreadSanitizer所能检测出的线程问题只占了一小部分。对于许多棘手的多线程问题，它们也无能为力。工具报告没问题并不确保代码没问题，要写出线程安全的代码，还是得多花点心思。