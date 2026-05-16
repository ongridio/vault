---
title: GitHub - Qihoo360/mysql-sniffer: mysql-sniffer is a network traffic analyzer tool for mysql, it is developed by Qihoo DBA and infrastructure team
source: https://github.com/Qihoo360/mysql-sniffer
kind: external
domain: observability
author: Qihoo
original_date: 2017-02-27
fetched_at: 2026-05-16
bookmark_title: Qihoo360/mysql-sniffer: mysql-sniffer is a network traffic analyzer tool for mysql, it is developed by Qihoo DBA and infrastructure team
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[github.com](https://github.com/Qihoo360/mysql-sniffer)
> 作者：Qihoo
> 原始日期：2017-02-27
> 抓取日期：2026-05-16

# GitHub - Qihoo360/mysql-sniffer: mysql-sniffer is a network traffic analyzer tool for mysql, it is developed by Qihoo DBA and infrastructure team

# MySQL Sniffer 中文介绍

MySQL Sniffer is a network traffic analyzer tool for MySQL, it is developed by Qihoo DBA and infrastructure team. This commandline tool captures and analyzes packets destined for a MySQL server or Client, and outputs them in a standard log format including access time, users, IP, database, query_time, rows number and query.

MySQL Sniffer also analyzer Atlas's network traffic. Atlas is a MySQL protocol-based database middleware project，github：https://github.com/Qihoo360/Atlas

- Certified to run on CentOS v6
- Commandline access to the server with root privileges

```
./mysql-sniffer -h
Usage mysql-sniffer [-d] -i eth0 -p 3306,3307,3308 -l /var/log/mysql-sniffer/ -e stderr
[-d] -i eth0 -r 3000-4000
-d daemon mode.
-s how often to split the log file(minute, eg. 1440). if less than 0, split log everyday
-i interface. Default to eth0
-p port, default to 3306. Multiple ports should be splited by ','. eg. 3306,3307
this option has no effect when -f is set.
-r port range, Don't use -r and -p at the same time
-l query log DIRECTORY. Make sure that the directory is accessible. Default to stdout.
-e error log FILENAME or 'stderr'. if set to /dev/null, runtime error will not be recorded
-f filename. use pcap file instead capturing the network interface
-w white list. dont capture the port. Multiple ports should be splited by ','.
-t truncation length. truncate long query if it's longer than specified length. Less than 0 means no truncation
-n keeping tcp stream count, if not set, default is 65536. if active tcp count is larger than the specified count, mysql-sniffer will remove the oldest one
```


```
git clone https://github.com/Qihoo360/mysql-sniffer
cd mysql-sniffer
mkdir proj
cd proj
cmake ../
make
cd bin/
```


glib2-devel(2.28.8)、libpcap-devel(1.4.0)、libnet-devel(1.1.6)

```
git clone git@github.com:Qihoo360/mysql-sniffer.git
cd mysql-sniffer
mkdir proj
cd proj
cmake ../
make
cd bin/
```


More MySQL Sniffer information, Atlas and some other technology please pay attention to our Hulk platform official account or QQ:104180820

Thanks for the contributions yihaoDeng and winkyao have made for MySQL Sniffer