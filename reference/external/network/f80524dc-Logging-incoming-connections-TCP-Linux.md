---
title: Logging incoming connections (TCP, Linux)
source: http://blog.dgunia.de/2015/09/04/logging-incoming-connections-tcp-linux/
kind: external
domain: network
author: Author Dominique
original_date: 2015-09-04
fetched_at: 2026-05-16
bookmark_title: Logging incoming connections (TCP, Linux) – TechBlog
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.dgunia.de](http://blog.dgunia.de/2015/09/04/logging-incoming-connections-tcp-linux/)
> 作者：Author Dominique
> 原始日期：2015-09-04
> 抓取日期：2026-05-16

# Logging incoming connections (TCP, Linux)

On a server that I am running MRTG showed me some spikes in incoming connections. It seems that about 800 incoming tcp connections were made within a few minutes. Nagios reported that there were so many connections that no further connections could be created. To examine the problem I wanted to log all incoming TCP connections. This can be done with a single command:

iptables -I INPUT -p tcp --syn -j LOG --log-prefix='[tcpconnections] '

It temporarily adds a rule to the firewall’s INPUT chain to write all incoming TCP packets that have the SYN flag set, i.e. that try to initiate a connection, into the system’s log with the prefix “[tcpconnections] “. Afterward you can see the connections e.g. in /var/log/syslog.

However now the syslog is quickly filled with entries if your server has a lot of TCP traffic. To separate these entries and write them into a different log, add a file “*tcpconnections.conf*” to “*/etc/rsyslog.d*“:

:msg,contains,"[tcpconnections] " /var/log/tcpconnections.log & ~

The first line copies the matching entries into the file */var/log/tcpconnections.log*. The second line discards the same entries so that they are not additionally copied into the main syslog. The ampersand means that another action should be applied to the filter in the previous line. A good overview of these parameters can be found in the RedHat documentation here:

Basic configuration of rsyslog

To activate the changes, restart rsyslog:

/etc/init.d/rsyslog restart

Now all connections are logged to *tcpconnections.log* but the file can get large. So it should be rotated. To do this create a file “*tcpconnections*” in */etc/logrotate.d* and write this into it:

/var/log/tcpconnections.log { daily rotate 12 compress missingok }

It will create a new log each day and keep the last 12 logs.