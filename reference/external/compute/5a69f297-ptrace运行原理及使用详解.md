---
title: ptrace运行原理及使用详解
source: https://blog.csdn.net/edonlii/article/details/8717029
kind: external
domain: compute
author: 成就一亿技术人
original_date: 2013-03-25
fetched_at: 2026-05-16
bookmark_title: ptrace运行原理及使用详解 - CSDN博客
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/edonlii/article/details/8717029)
> 作者：成就一亿技术人
> 原始日期：2013-03-25
> 抓取日期：2026-05-16

# ptrace运行原理及使用详解

你想过怎么实现对系统调用的拦截吗？你尝试过通过改变系统调用的参数来愚弄你的系统kernel吗？你想过调试器是如何使运行中的进程暂停并且控制它吗？

你可能会开始考虑怎么使用复杂的kernel编程来达到目的，那么，你错了。实际上Linux提供了一种优雅的机制来完成这些：ptrace系统函数。 ptrace提供了一种使父进程得以监视和控制其它进程的方式，它还能够改变子进程中的寄存器和内核映像，因而可以实现断点调试和系统调用的跟踪。

使用ptrace，你可以在用户层拦截和修改系统调用(sys call)

在这篇文章中，我们将学习如何拦截一个系统调用，然后修改它的参数。在本文的第二部分我们将学习更先进的技术：设置断点，插入代码到一个正在运行的程序中；我们将潜入到机器内部，偷窥和纂改进程的寄存器和数据段。


**基本知识**

**
**操作系统提供了一种标准的服务来让程序员实现对底层硬件和服务的控制（比如文件系统），叫做系统调用(system calls)。当一个程序需要作系统调用的时候，它将相关参数放进系统调用相关的寄存器，然后调用软中断0x80，这个中断就像一个让程序得以接触到内核模式的窗口，程序将参数和系统调用号交给内核，内核来完成系统调用的执行。

在i386体系中(本文中所有的代码都是面向i386体系)，系统调用号将放入%eax,它的参数则依次放入%ebx, %ecx, %edx, %esi 和 %edi。 比如，在以下的调用。

Write(2, “Hello”, 5)

的汇编形式大概是这样的

movl $4, %eax

movl $2, %ebx

movl $hello, %ecx

movl $5, %edx

int $0x80

这里的$hello指向的是标准字符串”Hello”。

那么，ptrace会在什么时候出现呢？在执行系统调用之前，内核会先检查当前进程是否处于被“跟踪”(traced)的状态。如果是的话，内核暂停当前进程并将控制权交给跟踪进程，使跟踪进程得以察看或者修改被跟踪进程的寄存器。

让我们来看一个例子，演示这个跟踪程序的过程：


```
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h> /* For constants
ORIG_EAX etc */
int main()
{
pid_t child;
long orig_eax;
child = fork();
if(child == 0) {
ptrace(PTRACE_TRACEME, 0, NULL, NULL);
execl("/bin/ls", "ls", NULL);
}
else {
wait(NULL);
orig_eax = ptrace(PTRACE_PEEKUSER,
child, 4 * ORIG_EAX,
NULL);
printf("The child made a "
"system call %ld ", orig_eax);
ptrace(PTRACE_CONT, child, NULL, NULL);
}
return 0;
}
```


运行这个程序，将会在输出ls命令的结果的同时，输出:

The child made a system call 11

说明：11是execve的系统调用号，这是该程序调用的第一个系统调用。

想知道系统调用号的详细内容，察看 /usr/include/asm/unistd.h。

在以上的示例中，父进程fork出了一个子进程，然后跟踪它。在调用exec函数之前，子进程用PTRACE_TRACEME作为第一个参数调用了ptrace函数，它告诉内核：让别人跟踪我吧！然后，在子进程调用了execve()之后，它将控制权交还给父进程。当时父进程正使用wait()函数来等待来自内核的通知，现在它得到了通知，于是它可以开始察看子进程都作了些什么，比如看看寄存器的值之类。

ptrace函数的参数

Ptrace有四个参数

long ptrace(enum __ptrace_request request,

pid_t pid,

void *addr,

void *data);


第一个参数决定了ptrace的行为与其它参数的使用方法，可取的值有:

出现系统调用之后，内核会将eax中的值（此时存的是系统调用号）保存起来，我们可以使用PTRACE_PEEKUSER作为ptrace的第一个参数来读到这个值。我们察看完系统调用的信息后，可以使用PTRACE_CONT作为ptrace的第一个参数，调用ptrace使子进程继续系统调用的过程。

第一个参数决定了ptrace的行为与其它参数的使用方法，可取的值有:

PTRACE_ME

PTRACE_PEEKTEXT

PTRACE_PEEKDATA

PTRACE_PEEKUSER

PTRACE_POKETEXT

PTRACE_POKEDATA

PTRACE_POKEUSER

PTRACE_GETREGS

PTRACE_GETFPREGS,

PTRACE_SETREGS

PTRACE_SETFPREGS

PTRACE_CONT

PTRACE_SYSCALL,

PTRACE_SINGLESTEP

PTRACE_DETACH


在下文中将对这些常量的用法进行说明。


**读取系统调用的参数**

通过将PTRACE_PEEKUSER作为ptrace 的第一个参数进行调用，可以取得与子进程相关的寄存器值。


先看下面这个例子



```
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>
#include <sys/syscall.h> /* For SYS_write etc */
int main()
{
pid_t child;
long orig_eax, eax;
long params[3];
int status;
int insyscall = 0;
child = fork();
if(child == 0) {
ptrace(PTRACE_TRACEME, 0, NULL, NULL);
execl("/bin/ls", "ls", NULL);
}
else {
while(1) {
wait(&status);
if(WIFEXITED(status))
break;
orig_eax = ptrace(PTRACE_PEEKUSER,
child, 4 * ORIG_EAX, NULL);
if(orig_eax == SYS_write) {
if(insyscall == 0) {
/* Syscall entry */
insyscall = 1;
params[0] = ptrace(PTRACE_PEEKUSER,
child, 4 * EBX,
NULL);
params[1] = ptrace(PTRACE_PEEKUSER,
child, 4 * ECX,
NULL);
params[2] = ptrace(PTRACE_PEEKUSER,
child, 4 * EDX,
NULL);
printf("Write called with "
"%ld, %ld, %ld ",
params[0], params[1],
params[2]);
}
else { /* Syscall exit */
eax = ptrace(PTRACE_PEEKUSER,
child, 4 * EAX, NULL);
printf("Write returned "
"with %ld ", eax);
insyscall = 0;
}
}
ptrace(PTRACE_SYSCALL,
child, NULL, NULL);
}
}
return 0;
}
```


这个程序的输出是这样的ppadala@linux:~/ptrace > ls

a.out dummy.s ptrace.txt

libgpm.html registers.c syscallparams.c

dummy ptrace.html simple.c

ppadala@linux:~/ptrace > ./a.out

Write called with 1, 1075154944, 48

a.out dummy.s ptrace.txt

Write returned with 48

Write called with 1, 1075154944, 59

libgpm.html registers.c syscallparams.c

Write returned with 59

Write called with 1, 1075154944, 30

dummy ptrace.html simple.c

Write returned with 30

以上的例子中我们跟踪了write系统调用，而ls命令的执行将产生三个write系统调用。使用PTRACE_SYSCALL作为ptrace的第一个参数，使内核在子进程做出系统调用或者准备退出的时候暂停它。这种行为与使用PTRACE_CONT，然后在下一个系统调用/进程退出时暂停它是等价的。

在前一个例子中，我们用PTRACE_PEEKUSER来察看write系统调用的参数。系统调用的返回值会被放入%eax。

wait函数使用status变量来检查子进程是否已退出。它是用来判断子进程是被ptrace暂停掉还是已经运行结束并退出。有一组宏可以通过status的值来判断进程的状态，比如WIFEXITED等，详情可以察看wait(2) man。





**读取寄存器的值**


如果你想在系统调用或者进程终止的时候读取它的寄存器，使用前面那个例子的方法是可以的，但是这是笨拙的方法。使用PRACE_GETREGS作为ptrace的第一个参数来调用，可以只需一次函数调用就取得所有的相关寄存器值。


获得寄存器值得例子如下：


```
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>
#include <sys/syscall.h>
int main()
{
pid_t child;
long orig_eax, eax;
long params[3];
int status;
int insyscall = 0;
struct user_regs_struct regs;
child = fork();
if(child == 0) {
ptrace(PTRACE_TRACEME, 0, NULL, NULL);
execl("/bin/ls", "ls", NULL);
}
else {
while(1) {
wait(&status);
if(WIFEXITED(status))
break;
orig_eax = ptrace(PTRACE_PEEKUSER,
child, 4 * ORIG_EAX,
NULL);
if(orig_eax == SYS_write) {
if(insyscall == 0) {
/* Syscall entry */
insyscall = 1;
ptrace(PTRACE_GETREGS, child,
NULL, ®s);
printf("Write called with "
"%ld, %ld, %ld ",
regs.ebx, regs.ecx,
regs.edx);
}
else { /* Syscall exit */
eax = ptrace(PTRACE_PEEKUSER,
child, 4 * EAX,
NULL);
printf("Write returned "
"with %ld ", eax);
insyscall = 0;
}
}
ptrace(PTRACE_SYSCALL, child,
NULL, NULL);
}
}
return 0;
}
```


这段代码与前面的例子是比较相似的，不同的是它使用了PTRACE_GETREGS。 其中的user_regs_struct结构是在<linux/user.h>中定义的。**单步**

ptrace提供了对子进程进行单步的功能。 ptrace(PTRACE_SINGLESTEP, …) 会使内核在子进程的每一条指令执行前先将其阻塞，然后将控制权交给父进程。下面的例子可以查出子进程当前将要执行的指令。为了便于理解，我用汇编写了这个受控程序，而不是让你为c的库函数到底会作那些系统调用而头痛。


以下是被控程序的代码 dummy1.s，使用gcc –o dummy1 dummy1.s来编译

.data

hello:

.string "hello world/n"

.globl main

main:

movl $4, %eax

movl $2, %ebx

movl $hello, %ecx

movl $12, %edx

int $0x80

movl $1, %eax

xorl %ebx, %ebx

int $0x80

ret


以下的程序则用来完成单步：


```
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>
#include <sys/syscall.h>
int main()
{
pid_t child;
const int long_size = sizeof(long);
child = fork();
if(child == 0) {
ptrace(PTRACE_TRACEME, 0, NULL, NULL);
execl("./dummy1", "dummy1", NULL);
}
else {
int status;
union u {
long val;
char chars[long_size];
}data;
struct user_regs_struct regs;
int start = 0;
long ins;
while(1) {
wait(&status);
if(WIFEXITED(status))
break;
ptrace(PTRACE_GETREGS,
child, NULL, ®s);
if(start == 1) {
ins = ptrace(PTRACE_PEEKTEXT,
child, regs.eip,
NULL);
printf("EIP: %lx Instruction "
"executed: %lx ",
regs.eip, ins);
}
if(regs.orig_eax == SYS_write) {
start = 1;
ptrace(PTRACE_SINGLESTEP, child,
NULL, NULL);
}
else
ptrace(PTRACE_SYSCALL, child,
NULL, NULL);
}
}
return 0;
}
```


程序的输出是这样的：

你可能需要察看Intel的用户手册来了解这些指令代码的意思。

更复杂的单步，比如设置断点，则需要很仔细的设计和更复杂的代码才可以实现。

在第一部分中我们已经看到ptrace怎么获取子进程的系统调用以及改变系统调用的参数。在这篇文章中，我们将要研究如何在子进程中设置断点和往运行中的程序里插入代码。实际上调试器就是用这种方法来设置断点和执行调试句柄。与前面一样，这里的所有代码都是针对i386平台的。

**附着在进程上**

在第一部分钟，我们使用ptrace(PTRACE_TRACEME, …)来跟踪一个子进程，如果你只是想要看进程是怎么进行系统调用和跟踪程序的，这个做法是不错的。但如果你要对运行中的进程进行调试，则需要使用 ptrace( PTRACE_ATTACH, ….)

当 ptrace( PTRACE_ATTACH, …)在被调用的时候传入了子进程的pid时， 它大体是与ptrace( PTRACE_TRACEME, …)的行为相同的，它会向子进程发送SIGSTOP信号，于是我们可以察看和修改子进程，然后使用 ptrace( PTRACE_DETACH, …)来使子进程继续运行下去。


下面是调试程序的一个简单例子。

```
int main()
{
int i;
for(i = 0;i < 10; ++i) {
printf("My counter: %d ", i);
sleep(2);
}
return 0;
}
```


将上面的代码保存为dummy2.c。按下面的方法编译运行：


gcc -o dummy2 dummy2.c

./dummy2 &


现在我们可以用下面的代码来附着到dummy2上。


```
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h> /* For user_regs_struct
etc. */
int main(int argc, char *argv[])
{
pid_t traced_process;
struct user_regs_struct regs;
long ins;
if(argc != 2) {
printf("Usage: %s <pid to be traced> ",
argv[0], argv[1]);
exit(1);
}
traced_process = atoi(argv[1]);
ptrace(PTRACE_ATTACH, traced_process,
NULL, NULL);
wait(NULL);
ptrace(PTRACE_GETREGS, traced_process,
NULL, ®s);
ins = ptrace(PTRACE_PEEKTEXT, traced_process,
regs.eip, NULL);
printf("EIP: %lx Instruction executed: %lx ",
regs.eip, ins);
ptrace(PTRACE_DETACH, traced_process,
NULL, NULL);
return 0;
}
```


上面的程序仅仅是附着在子进程上，等待它结束，并测量它的eip( 指令指针)然后释放子进程。

**设置断点**


调试器是怎么设置断点的呢？通常是将当前将要执行的指令替换成trap指令，于是被调试的程序就会在这里停滞，这时调试器就可以察看被调试程序的信息了。被调试程序恢复运行以后调试器会把原指令再放回来。这里是一个例子：


```
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>
const int long_size = sizeof(long);
void getdata(pid_t child, long addr,
char *str, int len)
{
char *laddr;
int i, j;
union u {
long val;
char chars[long_size];
}data;
i = 0;
j = len / long_size;
laddr = str;
while(i < j) {
data.val = ptrace(PTRACE_PEEKDATA, child,
addr + i * 4, NULL);
memcpy(laddr, data.chars, long_size);
++i;
laddr += long_size;
}
j = len % long_size;
if(j != 0) {
data.val = ptrace(PTRACE_PEEKDATA, child,
addr + i * 4, NULL);
memcpy(laddr, data.chars, j);
}
str[len] = '';
}
void putdata(pid_t child, long addr,
char *str, int len)
{
char *laddr;
int i, j;
union u {
long val;
char chars[long_size];
}data;
i = 0;
j = len / long_size;
laddr = str;
while(i < j) {
memcpy(data.chars, laddr, long_size);
ptrace(PTRACE_POKEDATA, child,
addr + i * 4, data.val);
++i;
laddr += long_size;
}
j = len % long_size;
if(j != 0) {
memcpy(data.chars, laddr, j);
ptrace(PTRACE_POKEDATA, child,
addr + i * 4, data.val);
}
}
int main(int argc, char *argv[])
{
pid_t traced_process;
struct user_regs_struct regs, newregs;
long ins;
/* int 0x80, int3 */
char code[] = {0xcd,0x80,0xcc,0};
char backup[4];
if(argc != 2) {
printf("Usage: %s <pid to be traced> ",
argv[0], argv[1]);
exit(1);
}
traced_process = atoi(argv[1]);
ptrace(PTRACE_ATTACH, traced_process,
NULL, NULL);
wait(NULL);
ptrace(PTRACE_GETREGS, traced_process,
NULL, ®s);
/* Copy instructions into a backup variable */
getdata(traced_process, regs.eip, backup, 3);
/* Put the breakpoint */
putdata(traced_process, regs.eip, code, 3);
/* Let the process continue and execute
the int 3 instruction */
ptrace(PTRACE_CONT, traced_process, NULL, NULL);
wait(NULL);
printf("The process stopped, putting back "
"the original instructions ");
printf("Press <enter> to continue ");
getchar();
putdata(traced_process, regs.eip, backup, 3);
/* Setting the eip back to the original
instruction to let the process continue */
ptrace(PTRACE_SETREGS, traced_process,
NULL, ®s);
ptrace(PTRACE_DETACH, traced_process,
NULL, NULL);
return 0;
}
```


上面的程序将把三个byte的内容进行替换以执行trap指令，等被调试进程停滞以后，我们把原指令再替换回来并把eip修改为原来的值。下面的图中演示了指令的执行过程。



|
|
|
|
1. 进程停滞后 |
2. 替换入trap指令 |
|
|
|
|
3.断点成功，控制权交给了调试器 |
4. 继续运行，将原指令替换回来并将eip复原 |


**ptrace的幕后工作**

那么，在使用ptrace的时候，内核里发生了声么呢？这里有一段简要的说明：

当一个进程调用了 ptrace( PTRACE_TRACEME, …)之后，内核为该进程设置了一个标记，注明该进程将被跟踪。内核中的相关原代码如下：

Source: arch/i386/kernel/ptrace.c

if (request == PTRACE_TRACEME) {

/* are we already being traced? */

if (current->ptrace & PT_PTRACED)

goto out;

/* set the ptrace bit in the process flags. */

current->ptrace |= PT_PTRACED;

ret = 0;

goto out;

}

一次系统调用完成之后，内核察看那个标记，然后执行trace系统调用（如果这个进程正处于被跟踪状态的话）。其汇编的细节可以在 arh/i386/kernel/entry.S中找到。

现在让我们来看看这个sys_trace()函数（位于 arch/i386/kernel/ptrace.c ）。它停止子进程，然后发送一个信号给父进程，告诉它子进程已经停滞，这个信号会激活正处于等待状态的父进程，让父进程进行相关处理。父进程在完成相关操作以后就调用ptrace( PTRACE_CONT, …)或者 ptrace( PTRACE_SYSCALL, …), 这将唤醒子进程，内核此时所作的是调用一个叫wake_up_process() 的进程调度函数。其他的一些系统架构可能会通过发送SIGCHLD给子进程来达到这个目的。

**小结：**

ptrace函数可能会让人们觉得很奇特，因为它居然可以检测和修改一个运行中的程序。这种技术主要是在调试器和系统调用跟踪程序中使用。它使程序员可以在用户级别做更多有意思的事情。已经有过很多在用户级别下扩展操作系统得尝试，比如UFO,一个用户级别的文件系统扩展，它使用ptrace来实现一些安全机制。

作者:

Pradeep Padala,

p_padala@yahoo.com

http://www.cise.ufl.edu/~ppadala