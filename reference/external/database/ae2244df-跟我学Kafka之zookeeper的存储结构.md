---
title: 跟我学Kafka之zookeeper的存储结构
source: https://www.jianshu.com/p/3e9cedc8ed03
kind: external
domain: database
original_date: 2016-02-18
fetched_at: 2026-05-16
bookmark_title: 跟我学Kafka之zookeeper的存储结构 - 简书
tags: [external, database]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.jianshu.com](https://www.jianshu.com/p/3e9cedc8ed03)
> 原始日期：2016-02-18
> 抓取日期：2026-05-16

# 跟我学Kafka之zookeeper的存储结构

# 一、zookeeper存储结构总图

当我们kafka启动运行以后，就会在zookeeper上初始化kafka相关数据，主要包括六大类：

- consumers
- admin
- config
- controller
- brokers
- controller_epoch

## 1、brokers节点结构说明

### 1.1 topic信息结构

/brokers/topics/[topic] :


存储某个topic的partitions所有分配信息:

```
Schema:
{
"version": "版本编号目前固定为数字1",
"partitions": {
"partitionId编号": [
同步副本组brokerId列表
],
"partitionId编号": [
同步副本组brokerId列表
],
.......
}
}
Example:
{
"version": 1,
"partitions": {
"0": [1, 2],
"1": [2, 1],
"2": [1, 2],
}
}
```


### 1.2 partitions信息

/brokers/topics/[topic]/partitions/[0...N] 其中[0..N]表示partition索引号


/brokers/topics/[topic]/partitions/[partitionId]/state

```
Schema:
{
"controller_epoch": 表示kafka集群中的中央控制器选举次数,
"leader": 表示该partition选举leader的brokerId,
"version": 版本编号默认为1,
"leader_epoch": 该partition leader选举次数,
"isr": [同步副本组brokerId列表]
}
Example:
{
"controller_epoch": 1,
"leader": 2,
"version": 1,
"leader_epoch": 0,
"isr": [2, 1]
}
```


### 1.3 broker信息

/brokers/ids/[0...N]


每个broker的配置文件中都需要指定一个数字类型的id(全局不可重复),此节点为临时znode(EPHEMERAL)

```
Schema:
{
"jmx_port": jmx端口号,
"timestamp": kafka broker初始启动时的时间戳,
"host": 主机名或ip地址,
"version": 版本编号默认为1,
"port": kafka broker的服务端端口号,由server.properties中参数port确定
}
Example:
{
"jmx_port": 5051,
"timestamp":"1403061000000"
"version": 1,
"host": "127.0.0.1",
"port": 8081
}
```


## 2、Controller_epoch

/controller_epoch -> int (epoch)


此值为一个数字,kafka集群中第一个broker第一次启动时为1，以后只要集群中center controller（中央控制器）所在broker变更或挂掉，就会重新选举新的center controller，每次center controller变更controller_epoch值就会 + 1;

## 3、Controller信息

/controller -> int (broker id of the controller)


存储center controller（中央控制器）所在kafka broker的信息。

```
Schema:
{
"version": 版本编号默认为1,
"brokerid": kafka集群中broker唯一编号,
"timestamp": kafka broker中央控制器变更时的时间戳
}
Example:
{
"version": 1,
"brokerid": 3,
"timestamp": "1403061802981"
}
```


这个的意思就说明，当前的Controller所在的Broker机器是哪台，变更时间是多少等。

## 4、Consumer信息

/consumers/[groupId]/ids/[consumerIdString]


每个consumer都有一个唯一的ID(consumerId可以通过配置文件指定,也可以由系统生成),此id用来标记消费者信息。

```
Schema:
{
"version": 版本编号默认为1,
"subscription": { //订阅topic列表},
"topic名称": consumer中topic消费者线程数
"pattern": "static",
"timestamp": "consumer启动时的时间戳"
}
```


### 4.1 Consumer offset信息

/consumers/[groupId]/offsets/[topic]/[partitionId] -> long (offset)


用来跟踪每个consumer目前所消费的partition中最大的offset。此znode为持久节点，可以看出offset跟group_id有关,以表明当消费者组(consumer group)中一个消费者失效，重新触发balance,其他consumer可以继续消费。