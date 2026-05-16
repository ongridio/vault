---
title: 教你如何在linux上装逼，shell中颜色的设置
source: http://www.cnblogs.com/ccorz/p/5523297.html
kind: external
domain: observability
author: Ccorz
original_date: 2016-05-24
fetched_at: 2026-05-16
bookmark_title: 教你如何在linux上装逼，shell中颜色的设置 - ccorz - 博客园
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/ccorz/p/5523297.html)
> 作者：Ccorz
> 原始日期：2016-05-24
> 抓取日期：2026-05-16

# 教你如何在linux上装逼，shell中颜色的设置

# 教你如何在linux上装逼，shell中颜色的设置

linux启动后环境变量加载的顺序为：etc/profile → /etc/profile.d/*.sh → ~/.bash_profile → ~/.bashrc → [/etc/bashrc]

想修改某用户登录后shell字体的颜色，可在~/.bashrc中添加PS1内容即可，以下是我机器的设置：

# .bashrc # User specific aliases and functions alias rm='rm -i' alias cp='cp -i' alias mv='mv -i' alias vi='vim' # Source global definitions if [ -f /etc/bashrc ]; then . /etc/bashrc fi PS1="\[\e[37;40m\][\[\e[32;40m\]\u\[\e[37;40m\]@\h\[\e[35;40m\]\W\[\e[0m\]]\\$ " export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });user=$(whoami); echo $(date "+%Y-%m-%d %H:%M:%S"):$user:`pwd`/:$msg ---- $(who am i); } >> /tmp/`hostname`.`whoami`.history-timestamp' ~

实际效果：

PS1的常用参数如下：

\d ：#代表日期，格式为weekday month date，例如："Mon Aug 1" \H ：#完整的主机名称 \h ：#仅取主机的第一个名字 \t ：#显示时间为24小时格式，如：HH：MM：SS \T ：#显示时间为12小时格式 \A ：#显示时间为24小时格式：HH：MM \u ：#当前用户的账号名称 \v ：#BASH的版本信息 \w ：#完整的工作目录名称 \W ：#利用basename取得工作目录名称，所以只会列出最后一个目录 \# ：#下达的第几个命令 \$ ：#提示字符，如果是root时，提示符为：# ，普通用户则为：$

颜色值设置： PS1中设置字符颜色的格式为：\[\e[F;Bm\]，其中“F“为字体颜色，编号为30-37，“B”为背景颜色，编号为40-47。颜色表如下

F B 30 40 黑色 31 41 红色 32 42 绿色 33 43 黄色 34 44 蓝色 35 45 紫红色 36 46 青蓝色 37 47 白色