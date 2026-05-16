---
title: Makefile 中命令的@,-@,+@符号 作用， 案例分析
source: https://blog.csdn.net/elfprincexu/article/details/51886620
kind: external
domain: observability
author: 成就一亿技术人
original_date: 2025-08-22
fetched_at: 2026-05-16
bookmark_title: Makefile 中命令的@,-@,+@符号 作用， 案例分析 - elfprincexu的专栏 - CSDN博客
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/elfprincexu/article/details/51886620)
> 作者：成就一亿技术人
> 原始日期：2025-08-22
> 抓取日期：2026-05-16

# Makefile 中命令的@,-@,+@符号 作用， 案例分析

### Make/makefile中的加号+，减号-和at号@的含义


#### shell 命令

每个目标都可以具有与其关联的一系列 shell 命令，这些命令通常用来创建目标。此脚本中的每一条命令都必须以制表符开始。虽然任何目标都能够显示在相关性行上，但除非使用 :: 操作符，否则这些相关性中只有一个能够通过创建脚本来跟随。

如果命令行的第一个或前两个字符是 @ (at 符号)、-（连字符）和 +（加号）这几个符号之一或全部，那么将特别处理该命令，如下：


所以，简单的说就是：



#### 【make中命令行前面加上减号】

-means ignore the exit status of the command that is executed (normally, a non-zero exit status would stop that part of htat build) 通常情况下，Makefile在执行到某一条命令时，如果返回值不正常，就会推出当前make进程，通常结合 rm mkdir命令使用，（空文件或者文件不存在都会返回错误）

就是，忽略当前此行命令执行时候所遇到的错误。

而如果不忽略，make在执行命令的时候，如果遇到error，会退出执行的，加上减号的目的，是即便此行命令执行中出错，比如删除一个不存在的文件等，那么也不要管，继续执行make。



#### 【make中命令行前面加上at符号@】

@suppress the normal 'echo' of the command that is executed. 通常makefile执行到每一行时都会打印出该行信息，加上@符号后就可以不现实该条命令。 通常结合 @echo 来使用表示当前Makefile执行到哪里。

就是，在make执行时候，输出的信息中，不要显示此行命令。

而正常情况下，make执行过程中，都是会显示其所执行的任何的命令的。如果你不想要显示某行的命令，那么就在其前面加上@符号即可。



#### 【make中命令行前面加上加号+】

对于命令行前面加上加号+的含义，表明在使用 make -n 命令的时候，其他行都只是显示命令而不执行，只有+ 行的才会被执行。





## ----------------- 案例分析 ----------------


正常情况下，make会打印每条命令，然后再执行，这就叫做回声（echoing）。

`test: # 这是测试`


执行上面的规则，会得到下面的结果。

`$ make test # 这是测试`


在命令的前面加上@，就可以关闭回声。

`test: @# 这是测试`


现在再执行`make test`

，就不会有任何输出。

由于在构建过程中，需要了解当前在执行哪条命令，所以通常只在注释和纯显示的echo命令前面加上@。

`test: @# 这是测试 @echo TODO`