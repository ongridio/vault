---
title: Linux 系统性能监控工具 Tsar
source: https://www.hi-linux.com/posts/5198.html
kind: external
domain: network
original_date: 2016-05-18
fetched_at: 2026-05-16
bookmark_title: Linux 系统性能监控工具 Tsar - 运维之美
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.hi-linux.com](https://www.hi-linux.com/posts/5198.html)
> 原始日期：2016-05-18
> 抓取日期：2026-05-16

# Linux 系统性能监控工具 Tsar

### Tsar简介

- Tsar是淘宝自己开发的一个采集工具，主要用来收集服务器的系统信息(如cpu，io，mem，tcp等)，以及应用数据(如squid haproxy nginx等)。
- 收集到的数据存储在磁盘上，可以随时查询历史信息，输出方式灵活多样，另外支持将数据存储到mysql中，也可以将数据发送到nagios报警服务器。
- Tsar在展示数据时，可以指定模块，并且可以对多条信息的数据进行merge输出，带–live参数可以输出秒级的实时信息。
- Tsar能够比较方便的增加模块，只需要按照tsar的要求编写数据的采集函数和展现函数，就可以把自定义的模块加入到Tsar中。

**总体架构**

- Tsar是基于模块化设计的程序，程序有两部分组成：框架和模块。
- 框架程序源代码主要在src目录，而模块源代码主要在modules目录中。
- 框架提供对配置文件的解析，模块的加载，命令行参数的解析，应用模块的接口对模块原始数据的解析与输出。 模块提供接口给框架调用。
- Tsar依赖与cron每分钟执行采集数据，因此它需要系统安装并启用crond，安装后，tsar每分钟会执行
`tsar --cron`

来定时采集信息，并且记录到原始日志文件。

**Tsar的运行流程图**

主要执行流程

1.解析输入


根据用户的输入，初始化一些全局信息，如间隔时间，是否merge，是否指定模块，运行模式

2.读取配置文件信息


主要解析tsar的配置文件，如果include生效，则会解析include的配置文件

配置文件用来获得tsar需要加载的模块，输出方式，每一类输出方式包含的模块，和此输出方式的接收信息

如mod_cpu on代表采集cpu的信息

output_interface file,nagios表示向文件和nagios服务器发送采集信息和报警信息

3.加载相应模块


根据配置文件的模块开启关闭情况，将模块的动态库load到系统

4.tsar的三种运行模式


tsar在运行的时候有三种模式：

print模式仅仅输出指定的模块信息，默认显示最近一天的；

live模式是输出当前信息，可以精确到秒级

cron模式，此一般是crontab定时执行，每一分钟采集一次所有配置的模块信息，并将数据写入原始文件，在cron运行的时候 会判断是否配置输出到db或者nagios，如果配置则将相应格式的数据输出到对应接口。

5.释放资源


程序最后，释放动态库，程序结束

项目地址: https://github.com/alibaba/tsar

### Tsar安装

**从github上检出代码**

1 | $ git clone git://github.com/alibaba/tsar.git |

**从github上下载源码**

1 | $ wget -O tsar.zip https://github.com/alibaba/tsar/archive/master.zip --no-check-certificate |

**安装后生成的文件**

Tsar配置文件路径：/etc/tsar/tsar.conf，tsar的采集模块和输出的具体配置；


定时任务配置:/etc/cron.d/tsar，负责每分钟调用tsar执行采集任务；

日志文件轮转配置:/etc/logrotate.d/tsar，每个月会把tsar的本地存储进行轮转；

模块路径：/usr/local/tsar/modules，各个模块的动态库so文件；

### Tsar配置

**Tsar配置文件介绍**

- 定时任务配置

1 | $ cat /etc/cron.d/tsar |

如上所示，`/etc/cron.d/tsar`

里面负责每分钟以root用户的角色调用tsar命令来执行数据采集。

- 日志文件轮转

1 | $ cat /etc/logrotate.d/tsar |

在日志文件轮转配置中，每个月会把tsar的本地存储进行轮转，此外这里也设定了数据在`/var/log/tsar.data`

下

- 配置文件

`/etc/tsar/tsar.conf`

负责tsar的采集模块和输出的具体配置；在这里配置启用哪些模块，输出等内容。

1 | $ cat /etc/tsar/tsar.conf |

**常用参数说明**

debug_level 指定tsar的运行级别，主要用来调试使用


mod_xxx on/off 开启指定模块

out_interface 设置输出类型，支持file，nagios，db

out_stdio_mod 设置用户终端默认显示的模块

output_db_mod 设置哪些模块输出到数据库

output_db_addr 数据库的ip和端口

output_nagios_mod 设置哪些模块输出到nagios

include 支持include配置，主要用来加载用户的自定义模块

cycle_time 指定上报的间隔时间，由于tsar每一分钟采集一次，上报时会判断是否符合时间间隔，如设置300的话，则在0，5等整点分钟会上报nagios

threshold 设置某个要报警项的阀值,前面是模块和要监控的具体名称，后面的四个数据代表报警的范围，warn和critical的范围

- 自定义模块配置文件

`/etc/tsar/conf.d/`

这个目录下是用户的自定义模块配置文件,配置基本在用户开发自定义模块时确定，主要包含模块的开启，输出类型和报警范围

### Tsar使用介绍

在Tsar的使用中，可以参考下面的帮助信息，完成对应的监控。

1 | $ tsar -h |

- tsar命令行主要担负显示历史数据和实时数据的功能，因此有控制展示模块和格式化输出的参数，默认不带任何参数/选项的情况下，tsar打印汇总信息。
- tsar命令行主要显示给人看的，所以数据展示中都进行了k/m/g等的进位。
- tsar命令会在显示20行数据后再次打印各个列的列头，以利于用户理解数据的含义。
- tsar的列头信息包括2行，第一行为模块名，第二行为列名。
- tsar输出最后会作min/avg/max的汇总统计，统计所展示中的最小/平均/最大数据。

### Tsar使用实例

#### Tsar监控系统

查看可用的模块列表

1 | $ tsar -L |

查看指定模块的运行状况,模块是指`tsar -L`

列出来的名称。如查看CPU运行情况

1 | $ tsar --cpu |

查看实时数据

1 | $ tsar -l |

显示1天内的历史汇总(summury)信息，以默认5分钟为间隔

1 | $ tsar |

以1秒钟为间隔，实时打印tsar的概述数据

1 | $ tsar -i 1 -l |

tsar cpu监控

使用参数`-–cpu`

可以监控系统的cpu，参数user表示用户空间cpu, sys内核空间cpu使用情况，wait是IO对应的cpu使用情况，hirq,sirq分别是硬件中断，软件中断的使用情况，util是系统使用cpu的总计情况。

1 | $ tsar --cpu |

显示一天内的cpu和内存历史数据，以1分钟为间隔

1 | $ tsar --cpu --mem -i 1 |

显示一天内cpu的历史信息，以1分钟为间隔

1 | $ tsar --cpu -i 1 |

tsar监控虚拟内存和load情况

1 | $ tsar --swap --load |

tsar监控内存使用情况

1 | $ tsar --mem |

以2秒钟为间隔，实时打印mem的数据

1 | $ tsar --live --mem -i 2 |

tsar监控io使用情况

1 | $ tsar --io |

tsar监控网络监控统计

1 | $ tsar --traffic |

1 | $ tsar --tcp --udp -d 1 |

tsar监控查看系统tcp连接情况，5秒刷新一次

1 | $ tsar --tcp -l 5 |

tsar检查告警信息

查看最后一次tsar的提醒信息,这里包括了系统的cpu,io的告警情况。

1 | $ tsar --check --cpu --io |

tsar历史数据回溯

通过参数`-d 2`

可以查出两天前到现在的数据，`-i 1`

表示以每次1分钟作为采集显示。

1 | $ tsar -d 2 -i 1 |

tsar查看指定日期的数据

1 | $ tsar --load -d 20160518 #指定日期,格式YYYYMMDD |

tsar查看所有字段

1 | $ tsar --mem -D |

查看fstab指定挂在的系统目录的使用情况 ，`-I`

指定查看某个目录

1 | $ tsar --partition -I / |

#### Tsar监控应用

Tsar默认支持的模块,如下

1 | $ ls /usr/local/tsar/modules |

默认安装完后，只启用了系统相关的模块。如要监控应用就需手动启用相应模块，以Nginx为例

1 | $ vim /etc/tsar/tsar.conf |

验证Nginx模块是否启用

1 | $ tsar -L|grep nginx |

配置Nginx

该配置主要是为nginx开启status统计页面,给tsar提供http数据。Tsar统计的原理是通过获取status页面的输出结果，并对输出内容进行统计和计算得出的结果。而且其获取状态页的url默认是http://127.0.0.1/nginx_status ，所以在nginx上你必须有如下的配置

1 | location /nginx_status { |

注：以上的url并非不能更改，可以修改环境变量实现。其自带的几个环境变量如下。

1 | export NGX_TSAR_HOST=192.168.0.1 |

监控Nginx状态

1 | $ tsar --nginx -l -i 2 |

### 参考文档

http://www.google.com

http://code.taobao.org/p/tsar/wiki/index/

http://blog.csdn.net/Road_long/article/details/47959221

http://blog.itpub.net/22664653/viewspace-1273519/

http://www.361way.com/tsar-nginx/2308.html