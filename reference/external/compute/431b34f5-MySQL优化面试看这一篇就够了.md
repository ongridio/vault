---
title: MySQL优化/面试，看这一篇就够了
source: https://juejin.im/post/5c2c53396fb9a04a053fc7fe
kind: external
domain: compute
author: 程序员乔戈里
original_date: 2019-01-02
fetched_at: 2026-05-16
bookmark_title: MySQL优化/面试，看这一篇就够了 - 掘金
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[juejin.im](https://juejin.im/post/5c2c53396fb9a04a053fc7fe)
> 作者：程序员乔戈里
> 原始日期：2019-01-02
> 抓取日期：2026-05-16

# MySQL优化/面试，看这一篇就够了

# 本文概要

# 概述

## 为什么要优化

- 系统的吞吐量瓶颈往往出现在数据库的访问速度上
- 随着应用程序的运行，数据库的中的数据会越来越多，处理时间会相应变慢
- 数据是存放在磁盘上的，读写速度无法和内存相比

## 如何优化

- 设计数据库时：数据库表、字段的设计，存储引擎
- 利用好MySQL自身提供的功能，如索引等
- 横向扩展：MySQL集群、负载均衡、读写分离
- SQL语句的优化（收效甚微）

# 字段设计

字段类型的选择，设计规范，范式，常见设计案例


## 原则：尽量使用整型表示字符串

### 存储IP

`INET_ATON(str)`

，address to number

`INET_NTOA(number)`

，number to address

### MySQL内部的枚举类型（单选）和集合（多选）类型

但是因为维护成本较高因此不常使用，使用**关联表**的方式来替代`enum`


## 原则：定长和非定长数据类型的选择

decimal不会损失精度，存储空间会随数据的增大而增大。double占用固定空间，较大数的存储会损失精度。非定长的还有varchar、text


### 金额

对数据的精度要求较高，小数的运算和存储存在精度问题（不能将所有小数转换成二进制）


### 定点数decimal

`price decimal(8,2)`

有2位小数的定点数，定点数支持很大的数（甚至是超过`int,bigint`

存储范围的数）

### 小单位大数额避免出现小数

元->分

### 字符串存储

定长`char`

，非定长`varchar、text`

（上限65535，其中`varchar`

还会消耗1-3字节记录长度，而`text`

使用额外空间记录长度）

## 原则：尽可能选择小的数据类型和指定短的长度

## 原则：尽可能使用 not null

非`null`

字段的处理要比`null`

字段的处理高效些！且不需要判断是否为`null`

。

`null`

在MySQL中，不好处理，存储需要额外空间，运算也需要特殊的运算符。如`select null = null`

和`select null <> null`

（`<>`

为不等号）有着同样的结果，只能通过`is null`

和`is not null`

来判断字段是否为`null`

。

如何存储？MySQL中每条记录都需要额外的存储空间，表示每个字段是否为`null`

。因此通常使用特殊的数据进行占位，比如`int not null default 0`

、`string not null default ‘’`


## 原则：字段注释要完整，见名知意

## 原则：单表字段不宜过多

二三十个就极限了

## 原则：可以预留字段

在使用以上原则之前首先要满足业务需求


# 关联表的设计

外键

`foreign key`

只能实现一对一或一对多的映射

## 一对多

使用外键

## 多对多

单独新建一张表将多对多拆分成两个一对多

## 一对一

如商品的基本信息（`item`

）和商品的详细信息（`item_intro`

），通常使用相同的主键或者增加一个外键字段（`item_id`

）

# 范式 Normal Format

数据表的设计规范，一套越来越严格的规范体系（如果需要满足N范式，首先要满足N-1范式）。N


## 第一范式1NF：字段原子性

字段原子性，字段不可再分割。

关系型数据库，默认满足第一范式


注意比较容易出错的一点，在一对多的设计中使用逗号分隔多个外键，这种方法虽然存储方便，但不利于维护和索引（比如查找带标签`java`

的文章）

## 第二范式：消除对主键的部分依赖

即在表中加上一个与业务逻辑无关的字段作为主键


主键：可以唯一标识记录的字段或者字段集合。

| course_name | course_class | weekday（周几） | course_teacher |
|---|---|---|---|
| MySQL | 教育大楼1525 | 周一 | 张三 |
| Java | 教育大楼1521 | 周三 | 李四 |
| MySQL | 教育大楼1521 | 周五 | 张三 |

依赖：A字段可以确定B字段，则B字段依赖A字段。比如知道了下一节课是数学课，就能确定任课老师是谁。于是**周几**和**下一节课**和就能构成复合主键，能够确定去哪个教室上课，任课老师是谁等。但我们常常增加一个`id`

作为主键，而消除对主键的部分依赖。

对主键的部分依赖：某个字段依赖复合主键中的一部分。

解决方案：新增一个独立字段作为主键。

## 第三范式：消除对主键的传递依赖

传递依赖：B字段依赖于A，C字段又依赖于B。比如上例中，任课老师是谁取决于是什么课，是什么课又取决于主键`id`

。因此需要将此表拆分为两张表日程表和课程表（独立数据独立建表）：

| id | weekday | course_class | course_id |
|---|---|---|---|
| 1001 | 周一 | 教育大楼1521 | 3546 |

| course_id | course_name | course_teacher |
|---|---|---|
| 3546 | Java | 张三 |

这样就减少了数据的冗余（即使周一至周日每天都有Java课，也只是`course_id:3546`

出现了7次）

# 存储引擎选择

早期问题：如何选择MyISAM和Innodb？

现在不存在这个问题了，Innodb不断完善，从各个方面赶超MyISAM，也是MySQL默认使用的。


存储引擎Storage engine：MySQL中的数据、索引以及其他对象是如何存储的，是一套文件系统的实现。

## 功能差异

`show engines`


| Engine | Support | Comment |
|---|---|---|
| InnoDB | DEFAULT | Supports transactions, row-level locking, and foreign keys |
| MyISAM | YES | MyISAM storage engine |

## 存储差异

| MyISAM | Innodb | |
|---|---|---|
| 文件格式 | 数据和索引是分别存储的，数据`.MYD` ，索引`.MYI` |
数据和索引是集中存储的，`.ibd` |
| 文件能否移动 | 能，一张表就对应`.frm` 、`MYD` 、`MYI` 3个文件 |
否，因为关联的还有`data` 下的其它文件 |
| 记录存储顺序 | 按记录插入顺序保存 | 按主键大小有序插入 |
空间碎片（删除记录并`flush table 表名` 之后，表文件大小不变） |
产生。定时整理：使用命令`optimize table 表名` 实现 |
不产生 |
| 事务 | 不支持 | 支持 |
| 外键 | 不支持 | 支持 |
| 锁支持（锁是避免资源争用的一个机制，MySQL锁对用户几乎是透明的） | 表级锁定 | 行级锁定、表级锁定，锁定力度小并发能力高 |

锁扩展

表级锁（

`table-level lock`

）：`lock tables <table_name1>,<table_name2>... read/write`

，`unlock tables <table_name1>,<table_name2>...`

。其中`read`

是共享锁，一旦锁定任何客户端都不可读；`write`

是独占/写锁，只有加锁的客户端可读可写，其他客户端既不可读也不可写。锁定的是一张表或几张表。行级锁（

`row-level lock`

）：锁定的是一行或几行记录。共享锁：`select * from <table_name> where <条件> LOCK IN SHARE MODE;`

，对查询的记录增加共享锁；`select * from <table_name> where <条件> FOR UPDATE;`

，对查询的记录增加排他锁。这里值得注意的是：`innodb`

的行锁，其实是一个子范围锁，依据条件锁定部分范围，而不是就映射到具体的行上，因此还有一个学名：间隙锁。比如`select * from stu where id < 20 LOCK IN SHARE MODE`

会锁定`id`

在`20`

左右以下的范围，你可能无法插入`id`

为`18`

或`22`

的一条新纪录。

## 选择依据

如果没有特别的需求，使用默认的`Innodb`

即可。

MyISAM：以读写插入为主的应用程序，比如博客系统、新闻门户网站。

Innodb：更新（删除）操作频率也高，或者要保证数据的完整性；并发量高，支持事务和外键保证数据完整性。比如OA自动化办公系统。

# 索引

关键字与数据的映射关系称为索引（==包含关键字和对应的记录在磁盘中的地址==）。关键字是从数据当中提取的用于标识、检索数据的特定内容。


## 索引检索为什么快？

- 关键字相对于数据本身，==数据量小==
- 关键字是==有序==的，二分查找可快速确定位置

图书馆为每本书都加了索引号（类别-楼层-书架）、字典为词语解释按字母顺序编写目录等都用到了索引。

## MySQL中索引类型


普通索引（`key`

），唯一索引（`unique key`

），主键索引（`primary key`

），全文索引（`fulltext key`

）

三种索引的索引方式是一样的，只不过对索引的关键字有不同的限制：

- 普通索引：对关键字没有限制
- 唯一索引：要求记录提供的关键字不能重复
- 主键索引：要求关键字唯一且不为null

## 索引管理语法

### 查看索引

`show create table 表名`

：

`desc 表名`


### 创建索引

#### 创建表之后建立索引

```
create TABLE user_index(
id int auto_increment primary key,
first_name varchar(16),
last_name VARCHAR(16),
id_card VARCHAR(18),
information text
);
-- 更改表结构
alter table user_index
-- 创建一个first_name和last_name的复合索引，并命名为name
add key name (first_name,last_name),
-- 创建一个id_card的唯一索引，默认以字段名作为索引名
add UNIQUE KEY (id_card),
-- 鸡肋，全文索引不支持中文
add FULLTEXT KEY (information);
```


`show create table user_index`

：

#### 创建表时指定索引

```
CREATE TABLE user_index2 (
id INT auto_increment PRIMARY KEY,
first_name VARCHAR (16),
last_name VARCHAR (16),
id_card VARCHAR (18),
information text,
KEY name (first_name, last_name),
FULLTEXT KEY (information),
UNIQUE KEY (id_card)
);
```


### 删除索引

根据索引名删除普通索引、唯一索引、全文索引：`alter table 表名 drop KEY 索引名`


```
alter table user_index drop KEY name;
alter table user_index drop KEY id_card;
alter table user_index drop KEY information;
```


删除主键索引：`alter table 表名 drop primary key`

（因为主键只有一个）。这里值得注意的是，如果主键自增长，那么不能直接执行此操作（自增长依赖于主键索引）：

需要取消自增长再行删除：

```
alter table user_index
-- 重新定义字段
MODIFY id int,
drop PRIMARY KEY
```


但通常不会删除主键，因为设计主键一定与业务逻辑无关。

## 执行计划explain

```
CREATE TABLE innodb1 (
id INT auto_increment PRIMARY KEY,
first_name VARCHAR (16),
last_name VARCHAR (16),
id_card VARCHAR (18),
information text,
KEY name (first_name, last_name),
FULLTEXT KEY (information),
UNIQUE KEY (id_card)
);
insert into innodb1 (first_name,last_name,id_card,information) values ('张','三','1001','华山派');
```


我们可以通过`explain selelct`

来分析SQL语句执行前的执行计划：

由上图可看出此SQL语句是按照主键索引来检索的。

执行计划是：当执行SQL语句时，首先会分析、优化，形成执行计划，在按照执行计划执行。

## 索引使用场景（重点）

### where

上图中，根据`id`

查询记录，因为`id`

字段仅建立了主键索引，因此此SQL执行可选的索引只有主键索引，如果有多个，最终会选一个较优的作为检索的依据。

```
-- 增加一个没有建立索引的字段
alter table innodb1 add sex char(1);
-- 按sex检索时可选的索引为null
EXPLAIN SELECT * from innodb1 where sex='男';
```


可以尝试在一个字段未建立索引时，根据该字段查询的效率，然后对该字段建立索引（

`alter table 表名 add index(字段名)`

），同样的SQL执行的效率，你会发现查询效率会有明显的提升（数据量越大越明显）。

### order by

当我们使用`order by`

将查询结果按照某个字段排序时，如果该字段没有建立索引，那么执行计划会将查询出的所有数据使用外部排序（将数据从硬盘分批读取到内存使用内部排序，最后合并排序结果），这个操作是很影响性能的，因为需要将查询涉及到的所有数据从磁盘中读到内存（如果单条数据过大或者数据量过多都会降低效率），更无论读到内存之后的排序了。

但是如果我们对该字段建立索引`alter table 表名 add index(字段名)`

，那么由于索引本身是有序的，因此直接按照索引的顺序和映射关系逐条取出数据即可。而且如果分页的，那么只用**取出索引表某个范围内的索引对应的数据**，而不用像上述那**取出所有数据**进行排序再返回某个范围内的数据。（从磁盘取数据是最影响性能的）

### join

对

`join`

语句匹配关系（`on`

）涉及的字段建立索引能够提高效率

### 索引覆盖

如果要查询的字段都建立过索引，那么引擎会直接在索引表中查询而不会访问原始数据（否则只要有一个字段没有建立索引就会做全表扫描），这叫索引覆盖。因此我们需要尽可能的在`select`

后==只写必要的查询字段==，以增加索引覆盖的几率。

这里值得注意的是不要想着为每个字段建立索引，因为优先使用索引的优势就在于其体积小。

## 语法细节（要点）

在满足索引使用的场景下（

`where/order by/join on`

或索引覆盖），索引也不一定被使用

### 字段要独立出现

比如下面两条SQL语句在语义上相同，但是第一条会使用主键索引而第二条不会。

```
select * from user where id = 20-1;
select * from user where id+1 = 20;
```


`like`

查询，不能以通配符开头

比如搜索标题包含`mysql`

的文章：

```
select * from article where title like '%mysql%';
```


这种SQL的执行计划用不了索引（`like`

语句匹配表达式以通配符开头），因此只能做全表扫描，效率极低，在实际工程中几乎不被采用。而一般会使用第三方提供的支持中文的全文索引来做。

但是 **关键字查询** 热搜提醒功能还是可以做的，比如键入`mysql`

之后提醒`mysql 教程`

、`mysql 下载`

、`mysql 安装步骤`

等。用到的语句是：

```
select * from article where title like 'mysql%';
```


这种`like`

是可以利用索引的（当然前提是`title`

字段建立过索引）。

### 复合索引只对第一个字段有效

建立复合索引：

```
alter table person add index(first_name,last_name);
```


其原理就是将索引先按照从`first_name`

中提取的关键字排序，如果无法确定先后再按照从`last_name`

提取的关键字排序，也就是说该索引表只是按照记录的`first_name`

字段值有序。

因此`select * from person where first_name = ?`

是可以利用索引的，而`select * from person where last_name = ?`

无法利用索引。

那么该复合索引的应用场景是什么？==组合查询==


比如对于`select * person from first_name = ? and last_name = ?`

，复合索引就比对`first_name`

和`last_name`

单独建立索引要高效些。很好理解，复合索引首先二分查找与`first_name = ?`

匹配的记录，再在这些记录中二分查找与`last_name`

匹配的记录，只涉及到一张索引表。而分别单独建立索引则是在`first_name`

索引表中二分找出与`first_name = ?`

匹配的记录，再在`last_name`

索引表中二分找出与`last_name = ?`

的记录，两者取交集。

### or，两边条件都有索引可用

一但有一边无索引可用就会导致整个SQL语句的全表扫描


### 状态值，不容易使用到索引

如性别、支付状态等状态值字段往往只有极少的几种取值可能，这种字段即使建立索引，也往往利用不上。这是因为，一个状态值可能匹配大量的记录，这种情况MySQL会认为利用索引比全表扫描的效率低，从而弃用索引。索引是随机访问磁盘，而全表扫描是顺序访问磁盘，这就好比有一栋20层楼的写字楼，楼底下的索引牌上写着某个公司对应不相邻的几层楼，你去公司找人，与其按照索引牌的提示去其中一层楼没找到再下来看索引牌再上楼，不如从1楼挨个往上找到顶楼。

## 如何创建索引

- 建立基础索引：在
`where、order by、join`

字段上建立索引。 - 优化，组合索引：基于业务逻辑
- 如果条件经常性出现在一起，那么可以考虑将多字段索引升级为==复合索引==
- 如果通过增加个别字段的索引，就可以出现==索引覆盖==，那么可以考虑为该字段建立索引
- 查询时，不常用到的索引，应该删除掉


## 前缀索引

语法：`index(field(10))`

，使用字段值的前10个字符建立索引，默认是使用字段的全部内容建立索引。

前提：前缀的标识度高。比如密码就适合建立前缀索引，因为密码几乎各不相同。

==实操的难度==：在于前缀截取的长度。

我们可以利用`select count(*)/count(distinct left(password,prefixLen));`

，通过从调整`prefixLen`

的值（从1自增）查看不同前缀长度的一个平均匹配度，接近1时就可以了（表示一个密码的前`prefixLen`

个字符几乎能确定唯一一条记录）

## 索引的存储结构

### BTree

btree（多路平衡查找树）是一种广泛应用于==磁盘上实现索引功能==的一种数据结构，也是大多数数据库索引表的实现。

以`add index(first_name,last_name)`

为例：

BTree的一个node可以存储多个关键字，node的大小取决于计算机的文件系统，因此我们可以通过减小索引字段的长度使结点存储更多的关键字。如果node中的关键字已满，那么可以通过每个关键字之间的子节点指针来拓展索引表，但是不能破坏结构的有序性，比如按照`first_name`

第一有序、`last_name`

第二有序的规则，新添加的`韩香`

就可以插到`韩康`

之后。`白起 < 韩飞 < 韩康 < 李世民 < 赵奢 < 李寻欢 < 王语嫣 < 杨不悔`

。这与二叉搜索树的思想是一样的，只不过二叉搜索树的查找效率是`log(2,N)`

（以2为底N的对数），而BTree的查找效率是`log(x,N)`

（其中x为node的关键字数量，可以达到1000以上）。

从`log(1000+,N)`

可以看出，少量的磁盘读取即可做到大量数据的遍历，这也是btree的设计目的。

### B+Tree聚簇结构

聚簇结构（也是在BTree上升级改造的）中，关键字和记录是存放在一起的。

在MySQL中，仅仅只有`Innodb`

的==主键索引为聚簇结构==，其它的索引包括`Innodb`

的非主键索引都是典型的BTree结构。

### 哈希索引

在索引被载入内存时，使用哈希结构来存储。

# 查询缓存

缓存

`select`

语句的查询结果

## 在配置文件中开启缓存

windows上是`my.ini`

，linux上是`my.cnf`


在`[mysqld]`

段中配置`query_cache_type`

：

- 0：不开启
- 1：开启，默认缓存所有，需要在SQL语句中增加
`select sql-no-cache`

提示来放弃缓存 - 2：开启，默认都不缓存，需要在SQL语句中增加
`select sql-cache`

来主动缓存（==常用==）

更改配置后需要重启以使配置生效，重启后可通过`show variables like ‘query_cache_type’;`

来查看：

```
show variables like 'query_cache_type';
query_cache_type DEMAND
```


## 在客户端设置缓存大小

通过配置项`query_cache_size`

来设置：

```
show variables like 'query_cache_size';
query_cache_size 0
set global query_cache_size=64*1024*1024;
show variables like 'query_cache_size';
query_cache_size 67108864
```


## 将查询结果缓存

`select sql_cache * from user;`


## 重置缓存

`reset query cache;`


## 缓存失效问题（大问题）

当数据表改动时，基于该数据表的任何缓存都会被删除。（表层面的管理，不是记录层面的管理，因此失效率较高）

## 注意事项

- 应用程序，不应该关心
`query cache`

的使用情况。可以尝试使用，但不能由`query cache`

决定业务逻辑，因为`query cache`

由DBA来管理。 - 缓存是以SQL语句为key存储的，因此即使SQL语句功能相同，但如果多了一个空格或者大小写有差异都会导致匹配不到缓存。

# 分区

一般情况下我们创建的表对应一组存储文件，使用`MyISAM`

存储引擎时是一个`.MYI`

和`.MYD`

文件，使用`Innodb`

存储引擎时是一个`.ibd`

和`.frm`

（表结构）文件。

当数据量较大时（一般千万条记录级别以上），MySQL的性能就会开始下降，这时我们就需要将数据分散到多组存储文件，==保证其单个文件的执行效率==。

最常见的分区方案是按`id`

分区，如下将`id`

的哈希值对10取模将数据均匀分散到10个`.ibd`

存储文件中：

```
create table article(
id int auto_increment PRIMARY KEY,
title varchar(64),
content text
)PARTITION by HASH(id) PARTITIONS 10
```


查看`data`

目录：

==服务端的表分区对于客户端是透明的==，客户端还是照常插入数据，但服务端会按照分区算法分散存储数据。


## MySQL提供的分区算法

==分区依据的字段必须是主键的一部分==，分区是为了快速定位数据，因此该字段的搜索频次较高应作为强检索字段，否则依照该字段分区毫无意义


### hash(field)

相同的输入得到相同的输出。输出的结果跟输入是否具有规律无关。==仅适用于整型字段==

### key(field)

和`hash(field)`

的性质一样，只不过`key`

是==处理字符串==的，比`hash()`

多了一步从字符串中计算出一个整型在做取模操作。

```
create table article_key(
id int auto_increment,
title varchar(64),
content text,
PRIMARY KEY (id,title) -- 要求分区依据字段必须是主键的一部分
)PARTITION by KEY(title) PARTITIONS 10
```


### range算法

是一种==条件分区==算法，按照数据大小范围分区（将数据使用某种条件，分散到不同的分区中）。

如下，按文章的发布时间将数据按照2018年8月、9月、10月分区存放：

```
create table article_range(
id int auto_increment,
title varchar(64),
content text,
created_time int, -- 发布时间到1970-1-1的毫秒数
PRIMARY KEY (id,created_time) -- 要求分区依据字段必须是主键的一部分
)charset=utf8
PARTITION BY RANGE(created_time)(
PARTITION p201808 VALUES less than (1535731199), -- select UNIX_TIMESTAMP('2018-8-31 23:59:59')
PARTITION p201809 VALUES less than (1538323199), -- 2018-9-30 23:59:59
PARTITION p201810 VALUES less than (1541001599) -- 2018-10-31 23:59:59
);
```


注意：条件运算符只能使用==less than==，这以为着较小的范围要放在前面，比如上述`p201808,p201819,p201810`

分区的定义顺序依照`created_time`

数值范围从小到大，不能颠倒。

```
insert into article_range values(null,'MySQL优化','内容示例',1535731180);
flush tables; -- 使操作立即刷新到磁盘文件
```


由于插入的文章的发布时间`1535731180`

小于`1535731199`

（`2018-8-31 23:59:59`

），因此被存储到`p201808`

分区中，这种算法的存储到哪个分区取决于数据状况。

### list算法

也是一种条件分区，按照列表值分区（`in (值列表)`

）。

```
create table article_list(
id int auto_increment,
title varchar(64),
content text,
status TINYINT(1), -- 文章状态：0-草稿，1-完成但未发布，2-已发布
PRIMARY KEY (id,status) -- 要求分区依据字段必须是主键的一部分
)charset=utf8
PARTITION BY list(status)(
PARTITION writing values in(0,1), -- 未发布的放在一个分区
PARTITION published values in (2) -- 已发布的放在一个分区
);
```


```
insert into article_list values(null,'mysql优化','内容示例',0);
flush tables;
```


## 分区管理语法

### range/list

#### 增加分区

前文中我们尝试使用`range`

对文章按照月份归档，随着时间的增加，我们需要增加一个月份：

```
alter table article_range add partition(
partition p201811 values less than (1543593599) -- select UNIX_TIMESTAMP('2018-11-30 23:59:59')
-- more
);
```


#### 删除分区

```
alter table article_range drop PARTITION p201808
```


注意：==删除分区后，分区中原有的数据也会随之删除！==

### key/hash

#### 新增分区

```
alter table article_key add partition partitions 4
```


#### 销毁分区

```
alter table article_key coalesce partition 6
```


`key/hash`

分区的管理不会删除数据，但是每一次调整（新增或销毁分区）都会将所有的数据重写分配到新的分区上。==效率极低==，最好在设计阶段就考虑好分区策略。

## 分区的使用

当数据表中的数据量很大时，分区带来的效率提升才会显现出来。

只有检索字段为分区字段时，分区带来的效率提升才会比较明显。因此，==分区字段的选择很重要==，并且==业务逻辑要尽可能地根据分区字段做相应调整==（尽量使用分区字段作为查询条件）。

# 水平分割和垂直分割

水平分割：通过建立结构相同的几张表分别存储数据

垂直分割：将经常一起使用的字段放在一个单独的表中，分割后的表记录之间是一一对应关系。


## 分表原因

- 为数据库减压
- 分区算法局限
- 数据库支持不完善（
`5.1`

之后`mysql`

才支持分区操作）

## id重复的解决方案

- 借用第三方应用如
`memcache、redis`

的`id`

自增器 - 单独建一张只包含
`id`

一个字段的表，每次自增该字段作为数据记录的`id`


# 集群

横向扩展：从根本上（单机的硬件处理能力有限）提升数据库性能 。由此而生的相关技术：==读写分离、负载均衡==


## 安装和配置主从复制

### 环境

`Red Hat Enterprise Linux Server release 7.0 (Maipo)`

（虚拟机）`mysql5.7`

（下载地址）

### 安装和配置

解压到对外提供的服务的目录（我自己专门创建了一个`/export/server`

来存放）

```
tar xzvf mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz -C /export/server
cd /export/server
mv mysql-5.7.23-linux-glibc2.12-x86_64 mysql
```


添加`mysql`

目录的所属组和所属者：

```
groupadd mysql
useradd -r -g mysql mysql
cd /export/server
chown -R mysql:mysql mysql/
chmod -R 755 mysql/
```


创建`mysql`

数据存放目录（其中`/export/data`

是我创建专门用来为各种服务存放数据的目录）

```
mkdir /export/data/mysql
```


初始化`mysql`

服务

```
cd /export/server/mysql
./bin/mysqld --basedir=/export/server/mysql --datadir=/export/data/mysql --user=mysql --pid-file=/export/data/mysql/mysql.pid --initialize
```


如果成功会显示

`mysql`

的`root`

账户的初始密码，记下来以备后续登录。如果报错缺少依赖，则使用`yum instally`

依次安装即可

配置`my.cnf`


```
vim /etc/my.cnf
[mysqld]
basedir=/export/server/mysql
datadir=/export/data/mysql
socket=/tmp/mysql.sock
user=mysql
server-id=10 # 服务id，在集群时必须唯一，建议设置为IP的第四段
port=3306
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd
[mysqld_safe]
log-error=/export/data/mysql/error.log
pid-file=/export/data/mysql/mysql.pid
#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```


将服务添加到开机自动启动

```
cp /export/server/mysql/support-files/mysql.server /etc/init.d/mysqld
```


启动服务

```
service mysqld start
```


配置环境变量，在`/etc/profile`

中添加如下内容

```
# mysql env
MYSQL_HOME=/export/server/mysql
MYSQL_PATH=$MYSQL_HOME/bin
PATH=$PATH:$MYSQL_PATH
export PATH
```


使配置即可生效

```
source /etc/profile
```


使用`root`

登录

```
mysql -uroot -p
# 这里填写之前初始化服务时提供的密码
```


登录上去之后，更改`root`

账户密码（我为了方便将密码改为root），否则操作数据库会报错

```
set password=password('root');
flush privileges;
```


设置服务可被所有远程客户端访问

```
use mysql;
update user set host='%' where user='root';
flush privileges;
```


这样就可以在宿主机使用

`navicat`

远程连接虚拟机linux上的mysql了

### 配置主从节点

#### 配置master

以`linux`

（`192.168.10.10`

）上的`mysql`

为`master`

，宿主机（`192.168.10.1`

）上的`mysql`

为`slave`

配置主从复制。

修改`master`

的`my.cnf`

如下

```
[mysqld]
basedir=/export/server/mysql
datadir=/export/data/mysql
socket=/tmp/mysql.sock
user=mysql
server-id=10
port=3306
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd
log-bin=mysql-bin # 开启二进制日志
expire-logs-days=7 # 设置日志过期时间，避免占满磁盘
binlog-ignore-db=mysql # 不使用主从复制的数据库
binlog-ignore-db=information_schema
binlog-ignore-db=performation_schema
binlog-ignore-db=sys
binlog-do-db=test #使用主从复制的数据库
[mysqld_safe]
log-error=/export/data/mysql/error.log
pid-file=/export/data/mysql/mysql.pid
#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```


重启`master`


```
service mysqld restart
```


登录`master`

查看配置是否生效（`ON`

即为开启，默认为`OFF`

）：

```
mysql> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin | ON |
+---------------+-------+
```


在`master`

的数据库中建立备份账号：`backup`

为用户名，`%`

表示任何远程地址，用户`back`

可以使用密码`1234`

通过任何远程客户端连接`master`


```
grant replication slave on *.* to 'backup'@'%' identified by '1234'
```


查看`user`

表可以看到我们刚创建的用户：

```
mysql> use mysql
mysql> select user,authentication_string,host from user;
+---------------+-------------------------------------------+-----------+
| user | authentication_string | host |
+---------------+-------------------------------------------+-----------+
| root | *81F5E21E35407D884A6CD4A731AEBFB6AF209E1B | % |
| mysql.session | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | localhost |
| mysql.sys | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | localhost |
| backup | *A4B6157319038724E3560894F7F932C8886EBFCF | % |
+---------------+-------------------------------------------+-----------+
```


新建`test`

数据库，创建一个`article`

表以备后续测试

```
CREATE TABLE `article` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`title` varchar(64) DEFAULT NULL,
`content` text,
PRIMARY KEY (`id`)
) CHARSET=utf8;
```


重启服务并刷新数据库状态到存储文件中（`with read lock`

表示在此过程中，客户端只能读数据，以便获得一个一致性的快照）

```
[root@zhenganwen ~]# service mysqld restart
Shutting down MySQL.... SUCCESS!
Starting MySQL. SUCCESS!
[root@zhenganwen mysql]# mysql -uroot -proot
mysql> flush tables with read lock;
Query OK, 0 rows affected (0.00 sec)
```


查看`master`

上当前的二进制日志和偏移量（记一下其中的`File`

和`Position`

）

```
mysql> show master status \G
*************************** 1. row ***************************
File: mysql-bin.000002
Position: 154
Binlog_Do_DB: test
Binlog_Ignore_DB: mysql,information_schema,performation_schema,sys
Executed_Gtid_Set:
1 row in set (0.00 sec)
```


`File`

表示实现复制功能的日志，即上图中的`Binary log`

；`Position`

则表示`Binary log`

日志文件的偏移量之后的都会同步到`slave`

中，那么在偏移量之前的则需要我们手动导入。

主服务器上面的任何修改都会保存在二进制日志Binary log里面，从服务器上面启动一个I/O thread（实际上就是一个主服务器的客户端进程），连接到主服务器上面请求读取二进制日志，然后把读取到的二进制日志写到本地的一个Realy log里面。从服务器上面开启一个SQL thread定时检查Realy log，如果发现有更改立即把更改的内容在本机上面执行一遍。

如果一主多从的话，这时主库既要负责写又要负责为几个从库提供二进制日志。此时可以稍做调整，将二进制日志只给某一从，这一从再开启二进制日志并将自己的二进制日志再发给其它从。或者是干脆这个从不记录只负责将二进制日志转发给其它从，这样架构起来性能可能要好得多，而且数据之间的延时应该也稍微要好一些

手动导入，从`master`

中导出数据

```
mysqldump -uroot -proot -hlocalhost test > /export/data/test.sql
```


将`test.sql`

中的内容在`slave`

上执行一遍。

#### 配置slave

修改`slave`

的`my.ini`

文件中的`[mysqld]`

部分

```
log-bin=mysql
server-id=1 #192.168.10.1
```


保存修改后重启`slave`

，`WIN+R`

->`services.msc`

->`MySQL5.7`

->重新启动

登录`slave`

检查`log_bin`

是否以被开启：

```
show VARIABLES like 'log_bin';
```


配置与`master`

的同步复制：

```
stop slave;
change master to
master_host='192.168.10.10', -- master的IP
master_user='backup', -- 之前在master上创建的用户
master_password='1234',
master_log_file='mysql-bin.000002', -- master上 show master status \G 提供的信息
master_log_pos=154;
```


启用`slave`

节点并查看状态

```
mysql> start slave;
mysql> show slave status \G
*************************** 1. row ***************************
Slave_IO_State: Waiting for master to send event
Master_Host: 192.168.10.10
Master_User: backup
Master_Port: 3306
Connect_Retry: 60
Master_Log_File: mysql-bin.000002
Read_Master_Log_Pos: 154
Relay_Log_File: DESKTOP-KUBSPE0-relay-bin.000002
Relay_Log_Pos: 320
Relay_Master_Log_File: mysql-bin.000002
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Replicate_Do_DB:
Replicate_Ignore_DB:
Replicate_Do_Table:
Replicate_Ignore_Table:
Replicate_Wild_Do_Table:
Replicate_Wild_Ignore_Table:
Last_Errno: 0
Last_Error:
Skip_Counter: 0
Exec_Master_Log_Pos: 154
Relay_Log_Space: 537
Until_Condition: None
Until_Log_File:
Until_Log_Pos: 0
Master_SSL_Allowed: No
Master_SSL_CA_File:
Master_SSL_CA_Path:
Master_SSL_Cert:
Master_SSL_Cipher:
Master_SSL_Key:
Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
Last_IO_Errno: 0
Last_IO_Error:
Last_SQL_Errno: 0
Last_SQL_Error:
Replicate_Ignore_Server_Ids:
Master_Server_Id: 10
Master_UUID: f68774b7-0b28-11e9-a925-000c290abe05
Master_Info_File: C:\ProgramData\MySQL\MySQL Server 5.7\Data\master.info
SQL_Delay: 0
SQL_Remaining_Delay: NULL
Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
Master_Retry_Count: 86400
Master_Bind:
Last_IO_Error_Timestamp:
Last_SQL_Error_Timestamp:
Master_SSL_Crl:
Master_SSL_Crlpath:
Retrieved_Gtid_Set:
Executed_Gtid_Set:
Auto_Position: 0
Replicate_Rewrite_DB:
Channel_Name:
Master_TLS_Version:
1 row in set (0.00 sec)
```


注意查看第4、14、15三行，若与我一致，表示

`slave`

配置成功

### 测试

关闭`master`

的读取锁定

```
mysql> unlock tables;
Query OK, 0 rows affected (0.00 sec)
```


向`master`

中插入一条数据

```
mysql> use test
mysql> insert into article (title,content) values ('mysql master and slave','record the cluster building succeed!:)');
Query OK, 1 row affected (0.00 sec)
```


查看`slave`

是否自动同步了数据

```
mysql> insert into article (title,content) values ('mysql master and slave','record the cluster building succeed!:)');
Query OK, 1 row affected (0.00 sec)
```


至此，主从复制的配置成功！：)

使用mysqlreplicate命令快速搭建 Mysql 主从复制

## 读写分离

读写分离是依赖于主从复制，而主从复制又是为读写分离服务的。因为主从复制要求`slave`

不能写只能读（如果对`slave`

执行写操作，那么`show slave status`

将会呈现`Slave_SQL_Running=NO`

，此时你需要按照前面提到的手动同步一下`slave`

）。

### 方案一、定义两种连接

就像我们在学JDBC时定义的`DataBase`

一样，我们可以抽取出`ReadDataBase,WriteDataBase implements DataBase`

，但是这种方式无法利用优秀的线程池技术如`DruidDataSource`

帮我们管理连接，也无法利用`Spring AOP`

让连接对`DAO`

层透明。

### 方案二、使用Spring AOP

如果能够使用`Spring AOP`

解决数据源切换的问题，那么就可以和`Mybatis`

、`Druid`

整合到一起了。

我们在整合`Spring1`

和`Mybatis`

时，我们只需写DAO接口和对应的`SQL`

语句，那么DAO实例是由谁创建的呢？实际上就是`Spring`

帮我们创建的，它通过我们注入的数据源，帮我们完成从中获取数据库连接、使用连接执行 `SQL`

语句的过程以及最后归还连接给数据源的过程。

如果我们能在调用DAO接口时根据接口方法命名规范（增`addXXX/createXXX`

、删`deleteXX/removeXXX`

、改`updateXXXX`

、查`selectXX/findXXX/getXX/queryXXX`

）动态地选择数据源（读数据源对应连接`master`

而写数据源对应连接`slave`

），那么就可以做到读写分离了。

#### 项目结构

#### 引入依赖

其中，为了方便访问数据库引入了`mybatis`

和`druid`

，实现数据源动态切换主要依赖`spring-aop`

和`spring-aspects`


```
<dependencies>
<dependency>
<groupId>org.mybatis</groupId>
<artifactId>mybatis-spring</artifactId>
<version>1.3.2</version>
</dependency>
<dependency>
<groupId>org.mybatis</groupId>
<artifactId>mybatis</artifactId>
<version>3.4.6</version>
</dependency>
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-core</artifactId>
<version>5.0.8.RELEASE</version>
</dependency>
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-aop</artifactId>
<version>5.0.8.RELEASE</version>
</dependency>
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-jdbc</artifactId>
<version>5.0.8.RELEASE</version>
</dependency>
<dependency>
<groupId>com.alibaba</groupId>
<artifactId>druid</artifactId>
<version>1.1.6</version>
</dependency>
<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<version>6.0.2</version>
</dependency>
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-context</artifactId>
<version>5.0.8.RELEASE</version>
</dependency>
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-aspects</artifactId>
<version>5.0.8.RELEASE</version>
</dependency>
<dependency>
<groupId>org.projectlombok</groupId>
<artifactId>lombok</artifactId>
<version>1.16.22</version>
</dependency>
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-test</artifactId>
<version>5.0.8.RELEASE</version>
</dependency>
<dependency>
<groupId>junit</groupId>
<artifactId>junit</artifactId>
<version>4.12</version>
</dependency>
</dependencies>
```


#### 数据类

```
package top.zhenganwen.mysqloptimize.entity;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Article {
private int id;
private String title;
private String content;
}
```


#### spring配置文件

其中`RoutingDataSourceImpl`

是实现动态切换功能的核心类，稍后介绍。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:context="http://www.springframework.org/schema/context"
xsi:schemaLocation="http://www.springframework.org/schema/beans http

[... 内容超长，已截断；完整原文见 source URL ...]
