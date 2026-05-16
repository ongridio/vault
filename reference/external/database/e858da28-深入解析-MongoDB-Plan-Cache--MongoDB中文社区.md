---
title: 深入解析 MongoDB Plan Cache | MongoDB中文社区
source: http://www.mongoing.com/archives/5624
kind: external
domain: database
author: 赵景波（Zbdba
original_date: 2018-06-11
fetched_at: 2026-05-16
tags: [external, database]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.mongoing.com](http://www.mongoing.com/archives/5624)
> 作者：赵景波（Zbdba
> 原始日期：2018-06-11
> 抓取日期：2026-05-16

# 深入解析 MongoDB Plan Cache | MongoDB中文社区

前段时间笔者遇到一个MongoBD Plan Cache的bug，于是深究了下MongoDB优化器相关源码。在这里分享给大家，一方面让大家知道MongoDB优化器工作原理，一方面就是避免踩坑。

首先帖一下笔者反馈给官方的bug地址：https://jira.mongodb.org/browse/SERVER-34785

注意目前官方仍在修复中，最新动态可参考：https://jira.mongodb.org/browse/SERVER-32452

接下来我们就进入正题，本文分为以下4个章节：

- 背景
- MongoDB生成执行计划是如何选择索引的
- 过滤符合条件的索引
- 选择合适的索引

- MongoDB Plan Cache机制
- 遇到该Bug如何处理
- 附录

## 1、背景

在2月26号下午2点37的时候，我们线上MongoDB性能出现拐点，开始出现大量的慢查，最终造成MongoDB异常夯住，最后通过重启MongoDB故障恢复。

首先我们知道是由于同类型的SQL突然改变执行计划选择了其他的索引，造成后续的SQL直接采用Cache中的执行计划全部成为慢查,最终导致实例夯住。那这里我们归纳为2个主要问题：

- MongoDB生成执行计划是如何选择索引的？
- MongoDB决定是否采用cache中的执行计划的条件是什么？

在有多个执行计划可选择的时候，MongoDB会选择一个最优的执行计划存放在cache中，MongoDB对同一类型的SQL（SQL基本一样，只有过滤条件具体的value不一样）会直接从cache中拿出执行计划，但是在采用cache之前也会进行判断是否继续采用cache中的执行计划。

那搞清楚上述两个问题之后，我们再结合我们线上的问题场景进行分析就可以得出最终的结论。

## 2. MongoDB生成执行计划是如何选择索引的

### 2.1. 过滤符合条件的索引

MongoDB会根据谓词条件也就是过滤条件来过滤合适的索引，比如以下SQL：

db.files.find({“operation.sha1Code”:”acca33867df96b97a05bdbd81f2aee13a50zbdba”,”operation.is_prefix”:true,”operation.des_url”:”sh/”})

它会选择带有operation.sha1Code、operation.is_prefix、operation.des_url字段的索引，不管是单个索引还是复合索引，所以在实际生产环境中选择了如下索引：

```
{
"v" : 1,
"key" : {
"operation.is_prefix" : 1
},
"name" : "operation.is_prefix_1",
"ns" : "fs.files",
"background" : true
},
{
"v" : 1,
"key" : {
"operation.des_url" : 1
},
"name" : "operation.des_url_1",
"ns" : "fs.files",
"background" : true
},
{
"v" : 1,
"key" : {
"operation.sha1Code" : 1
},
"name" : "operation.sha1Code_1",
"ns" : "fs.files",
"background" : true
}
```


但是上述索引谁最合适呢？

### 2.2. 选择合适的索引

前面我们提到了MongoDB会根据谓词条件选择多个索引，但是最终执行计划会选择一个索引，那MongoDB是怎么判断哪个索引能更快的得出该SQL的结果呢？

在MongoDB中，每个索引对应一种执行计划，在MongoDB中叫解决方案，在选择执行计划的时候每个执行计划都会扫描A次，这里的A次计算公式如下：

```
numWorks = std::max(static_cast(internalQueryPlanEvaluationWorks),
static_cast(fraction * collection->numRecords(txn)));
```


```
internalQueryPlanEvaluationWorks=10000
fraction=0.29
collection->numRecords(txn) 则为collection的总记录数
```


`那就是collection的总记录数*0.29如果比10000小就扫描10000次，如果比10000大那么就扫描collection数量*0.29次。`


那在每个执行计划扫描A次之后，MongoDB是如何选出最优的执行计划呢？

这里MongoDB有个计算score的公式：

```
double baseScore = 1;
size_t workUnits = stats->common.works;
double productivity =
static_cast(stats->common.advanced) / static_cast(workUnits);
const double epsilon = std::min(1.0 / static_cast(10 * workUnits), 1e-4);
double noFetchBonus = epsilon;
if (hasStage(STAGE_PROJECTION, stats) && hasStage(STAGE_FETCH, stats)) {
noFetchBonus = 0;
}
double noSortBonus = epsilon;
if (hasStage(STAGE_SORT, stats)) {
noSortBonus = 0;
}
double noIxisectBonus = epsilon;
if (hasStage(STAGE_AND_HASH, stats) || hasStage(STAGE_AND_SORTED, stats)) {
noIxisectBonus = 0;
}
double tieBreakers = noFetchBonus + noSortBonus + noIxisectBonus;
double score = baseScore + productivity + tieBreakers;
```


我们可以看到总的score是由baseScore+productivity+productivity构成的：

首先我们看看baseScore最开始赋值为1，后面再没有改变过，这里就不说了。

然后我们看看productivity的计算公式：

```
double productivity =
static_cast(stats->common.advanced) / static_cast(workUnits);
```


这里的common.advanced是每个索引扫描的时候是否能在collection拿到符合条件的记录，如果能拿到记录那么common.advanced就加1，workUnits则是总共扫描的次数

这里所有的执行计划扫描的次数都是一样的，所以common.advanced多的肯定productivity就大，MongoDB这样做主要是想选择能快速拿到SQL结果的索引。

然后我们再看tieBreakers，它是由noFetchBonus和noSortBonus和noIxisectBonus总和构成的。我们根据上述代码可以看到这三个值主要是控制MongoDB不要选择如下状态的：

```
STAGE_PROJECTION&&STAGE_FETCH（限定返回字段）
STAGE_SORT（避免排序）
STAGE_AND_HASH || STAGE_AND_SORTED（这个主要在交叉索引时产生）
```


它们的出现都比较影响性能，所以一旦它们出现，相应的值都会被设置成0.

这里的noFetchBonus和noSortBonus和noIxisectBonus的初始值都是epsilon，那么看看epsilon的计算公式：

```
const double epsilon = std::min(1.0 / static_cast(10 * workUnits), 1e-4);
```


这里workUnits就是每个索引的扫描次数，那么这里的意思就是取1.0 / static_cast(10 * workUnits)和1e-4中最小的值。

通过上述的计算score的公式，MongoDB就可以选择一个最优的执行计划，那这里还有一个起着决定性作用的一个状态，就是`IS_EOF`

。

在MongoDB扫描A次的时候，如果某个索引的命中数量小于A次，那它必然会提前扫描完，然后标志位状态为IS_EOF，这个时候MongoDB就会停止扫描，直接进入到计算score阶段，为什么要停止？

在我们执行SQL的时候，选择的索引针对于它的过滤条件是不是命中次数越少越好呀？这样我们最后扫描的collection记录是不是越少，这样总体执行时间就会越小。

所以在MongoDB计算score的时候，除了上述公式，还有以下条件：

```
if (statTrees[i]->common.isEOF) {
LOG(5) << "Adding +" << eofBonus << " EOF bonus to score." << endl;
score += 1;
}
```


`如果状态为IS_EOF则加一分，所以一般达到IS_EOF状态的索引都会被选中为最优执行计划。`


OK，说了这么多原理，我们以实际的生产SQL为例简单参照公式计算下，我们就以出问题的那个SQL来计算：

```
db.files.find({"operation.sha1Code":"acca33867df96b97a05bdbd81f2aee13a50zbdba","operation.is_prefix":true,"operation.des_url":"sh/"})
```


首先MongoDB会选择operation.sha1Code、operation.is_prefix、operation.des_url这三个字段的索引，然后扫描A次。OK，我们来计算一下A次是多少：

```
numWorks = std::max(static_cast(internalQueryPlanEvaluationWorks),
static_cast(fraction * collection->numRecords(txn)));
```


首先看看 0.29*collection总记录数量：

```
comos_temp:SECONDARY> db.files.find().count()
551033614
```


也就是551033614*0.29=159799748

那么A=159799748

然后我们再看看每个索引的命中数量：

```
comos_temp:SECONDARY> db.files.find({"operation.sha1Code":"acca33867df96b97a05bdbd81f2aee13a50zbdba"}).count()
539408
comos_temp:SECONDARY> db.files.find({"operation.is_prefix":true}).count()
463885621
comos_temp:SECONDARY> db.files.find({"operation.des_url":"sh/"}).count()
180999
```


这里我们很明显的看到operation.des_url字段的索引命中数量最小，只有180999，而且比A要小。所以在扫描180999次后，`operation.des_url率先到达IS_EOF状态`

，那么此刻将停止扫描，进入到算分的阶段。

这里计算score我就不一一计算了，因为这里很明显最后会选择operation.des_url字段的索引，因为它率先达到了`IS_EOF`

状态，说明它需要扫描的记录数量是最小的，最后的score的分值也是最高的，感兴趣的朋友可以计算下。

所以在线上我们看到这个问题SQL选择了operation.des_url字段的索引而不是像之前选择的operation.sha1Code字段的索引，这个时候我们就以为是MongoDB选错索引了，其实在这个SQL中，`MongoDB并没有选错索引，它选的很正确`

。

## 3、MongoDB Plan Cache机制

上面我们提到了MongoDB是如何选择索引最后生成最优的执行计划，那MongoDB会将最优的执行计划缓存到cache中，等待下次同样的SQL执行的时候会采用cache中的执行计划，但是在MongoDB中， 它并不是直接就采用了cache中的计划，而是有一些条件去判断，下面我们来看看MongoDB是如何判断的？

在`CachedPlanStage::pickBestPlan`

方法中，MongoDB会决定该SQL是继续采用cache中的执行计划，还是需要重新生成执行计划。

CachedPlanStage::pickBestPlan主要逻辑如下：

- 以Cache中执行计划的索引命中数量（works）*10=需要扫描的数量（B次）
- 继续用该索引扫描B次,扫描的过程中有如下几种情况：
- 索引每次扫描出来会去扫描collection，collection根据筛选条件如果能拿到记录，则返回advanced，如果返回的advanced累积次数超过101次，则继续用该执行计划。
- 索引扫描该字段的命中数量少于B次，则最终肯定会达到IS_EOF状态，这个时候还是继续用缓存中的执行计划。
- 如果扫描完了B次，但是发现返回advanced累积次数没有达到101次，则会重新生成执行计划
- 如果在扫描过程遇见错误，则会返回FAILURE，也会触发重新生成执行计划


OK，那我们来以实际故障举一个例子：

我们知道故障那一天从2月26号下午14点37分那条SQL重新生成执行计划后，就采用`operation.des_url`

字段索引而不是采用之前的`operation.sha1Code`

。并且后面的类似的SQL都是采用operation.des_url字段索引，这样导致大量的慢查。那这里我们可以看到后续的SQL其实都是直接采用cache中的执行计划，而没有重新生成执行计划。那为什么继续采用了cache中的执行计划呢？

我们实际看几条SQL（直接选用当时故障时候的慢查SQL）：

SQL1：

```
2018-02-26T14:39:11.049+0800 I COMMAND [conn3959231] command fs.files command: find { find: "files", filter: { operation.sha1Code: "e635838088b717ccfba83586375711c0a49zbdba", operation.is_prefix: true, operation.des_url: "astro/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.des_url: 1.0 } keysExamined:21366 docsExamined:21366 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:1146 nreturned:1 reslen:768 locks:{ Global: { acquireCount: { r: 2294 }, acquireWaitCount: { r: 1135 }, timeAcquiringMicros: { r: 21004892 } }, Database: { acquireCount: { r: 1147 } }, Collection: { acquireCount: { r: 1147 } } } protocol:op_query 50414ms
```


我们套用刚刚我们说的判断是否采用cache的逻辑：

1、首先需要扫描上次索引命中次数*10，那就是170146*10=1701460

2、这里我们从日志中可以看到该条sql operation.des_url: “astro/”的命中数量只有21366条，所以它是不是还没有扫描完1701460条就达到了IS_EOF状态了？

3、如果达到`IS_EOF`

状态就直接采用cache中的执行计划。

那我们可能会猜想是不是真的就是这个执行计划最优呢？我们上面已经详细说明了MongoDB是如何选择索引，其实我们简单的看看各个字段索引对应的命中数量我们就知道那个索引最优了。

```
comos_temp:SECONDARY> db.files.find({"operation.sha1Code": "e635838088b717ccfba83586375711c0a49zbdba"}).count()
1
comos_temp:SECONDARY> db.files.find({"operation.is_prefix":true}).count()
463885621
comos_temp:SECONDARY> db.files.find({"operation.des_url": "astro/"}).count()
23000（这里是现在的记录，当时是21366）
```


所以我们不用计算score是不是就能快速的得出结论，是不是采用operation.sha1Code字段的索引会更快一些，因为它只需要`扫描1条记录`

。

SQL2:

```
2018-02-26T14:40:08.200+0800 I COMMAND [conn3935510] command fs.files command: find { find: "files", filter: { operation.sha1Code: "46cdc6924ad8fc298f2cac0b3e853698583zbdba", operation.is_prefix: true, operation.des_url: "hebei/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.des_url: 1.0 } keysExamined:80967 docsExamined:80967 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:2807 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 5616 }, acquireWaitCount: { r: 2712 }, timeAcquiringMicros: { r: 53894897 } }, Database: { acquireCount: { r: 2808 } }, Collection: { acquireCount: { r: 2808 } } } protocol:op_query 127659ms
```


我们套用刚刚我们说的判断是否采用cache的逻辑：

1、首先需要扫描上次索引命中次数*10，那就是170146*10=1701460

2、这里我们从日志中可以看到该条sql operation.des_url: “hebei/”的命中数量只有80967条，所以它是不是还没有扫描完1701460条就达到了`IS_EOF`

状态了？

3、如果达到`IS_EOF`

状态就直接采用cache中的执行计划。

那我们可能会猜想是不是真的就是这个执行计划最优呢？我们上面已经详细说明了MongoDB是如何选择索引，其实我们简单的看看各个字段索引对应的命中数量我们就知道那个索引最优了。

```
comos_temp:SECONDARY> db.files.find({"operation.sha1Code": "46cdc6924ad8fc298f2cac0b3e853698583zbdba"}).count()
4
comos_temp:SECONDARY> db.files.find({"operation.is_prefix":true}).count()
463885621
comos_temp:SECONDARY> db.files.find({"operation.des_url": "hebei/"}).count()
85785（这里是现在的记录，当时是80967）
```


所以我们不用计算score是不是就能快速的得出结论，是不是采用operation.sha1Code字段的索引会更快一些，因为它只需要`扫描4条记录`

。

当然可以继续去找后面的慢查日志（当然一定是重启前的日志，重启后MongoDB会重新生成执行计划，这也是我们为什么重启MongoDB之后故障就恢复了）都是这种情况，所以这就是为什么有大量慢查，最终导致MongoDB异常夯住。

那这里我们是不是可以得出，`只要operation.des_url字段索引命中数量少于1701460条就会继续采用operation.des_url字段索引`

，`那这里其实就是MongoDB认为只要operation.des_url字段索引小于cache中该字段索引命中数量*10就认为这个索引是最优的`

，但是之前cache中的最优执行计划是建立在operation.sha1Code字段索引命中数量远大于operation.des_url字段索引命中数量的基础上。当时那个引发故障的SQL各个字段索引命中数量如下：

```
comos_temp:SECONDARY> db.files.find({"operation.sha1Code":"acca33867df96b97a05bdbd81f2aee13a50zbdba"}).count()
539408
comos_temp:SECONDARY> db.files.find({"operation.is_prefix":true}).count()
463885621
comos_temp:SECONDARY> db.files.find({"operation.des_url":"sh/"}).count()
180999
```


但是我们知道，operation.sha1Code字段索引命中数量是会变化的，随着具体的value不一样，其命中数量也不会一样。除了这条引发故障的SQL之外，其他SQL的operation.sha1Code字段命中索引数量都非常小，有的甚至只有一条。`那这里很明显MongoDB在cache中只去根据cache中执行计划的相关索引来进行判断是不合理的。`


当然这里还有一部分原因就是本身我们的collection有5亿条记录，那这个也是放大这次故障的一个主要原因。

那这里我们是不是想到了后面只要出现这条SQL就会导致慢查最终导致故障？

是的，确实是会导致慢查，但是不一定会导致故障，因为早在2017年的时候该SQL就出现了好几次。但是最终都自己恢复了，我们来看看是什么原因。

在2017-11-24T09:22:42的时候也执行了该SQL，不同的是operation.des_url 是 “sd/”,不过这并不影响，可以看到该sql出现，马上改变执行计划，开始采用operation.des_url索引，然后后面都是一些慢查了。

```
2017-11-24T09:22:42.767+0800 I COMMAND [conn3124017] command fs.files command: find { find: "files", filter: { operation.sha1Code: "acca33867df96b97a05bdbd81f2aee13a50zbdba", operation.is_prefix: true, operation.des_url: "sd/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.des_url: 1.0 } keysExamined:103934 docsExamined:103934 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:7923 nreturned:1 reslen:719 locks:{ Global: { acquireCount: { r: 15848 }, acquireWaitCount: { r: 860 }, timeAcquiringMicros: { r: 2941600 } }, Database: { acquireCount: { r: 7924 } }, Collection: { acquireCount: { r: 7924 } } } protocol:op_query 79379ms
2017-11-24T09:22:42.767+0800 I COMMAND [conn3080461] command fs.files command: find { find: "files", filter: { operation.sha1Code: "acca33867df96b97a05bdbd81f2aee13a50zbdba", operation.is_prefix: true, operation.des_url: "sd/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.des_url: 1.0 } keysExamined:103934 docsExamined:103934 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:7781 nreturned:1 reslen:719 locks:{ Global: { acquireCount: { r: 15564 }, acquireWaitCount: { r: 671 }, timeAcquiringMicros: { r: 2370524 } }, Database: { acquireCount: { r: 7782 } }, Collection: { acquireCount: { r: 7782 } } } protocol:op_query 64711ms
2017-11-24T09:24:05.957+0800 I COMMAND [conn3032629] command fs.files command: find { find: "files", filter: { operation.sha1Code: "3a85283ed90820d65174572f566026d8d1azbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:0 docsExamined:0 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8762 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 17526 }, acquireWaitCount: { r: 1000 }, timeAcquiringMicros: { r: 16267930 } }, Database: { acquireCount: { r: 8763 } }, Collection: { acquireCount: { r: 8763 } } } protocol:op_query 70696ms
2017-11-24T09:24:05.963+0800 I COMMAND [conn3038640] command fs.files command: find { find: "files", filter: { operation.sha1Code: "e84012269998d1f076f3d93cde43b1726fezbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:0 docsExamined:0 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8666 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 17334 }, acquireWaitCount: { r: 864 }, timeAcquiringMicros: { r: 14671764 } }, Database: { acquireCount: { r: 8667 } }, Collection: { acquireCount: { r: 8667 } } } protocol:op_query 58881ms
2017-11-24T09:24:06.006+0800 I COMMAND [conn3033497] command fs.files command: find { find: "files", filter: { operation.sha1Code: "19576f5a3b1ad0406080e15a4513d17badbzbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:0 docsExamined:0 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8806 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 17614 }, acquireWaitCount: { r: 1054 }, timeAcquiringMicros: { r: 16842848 } }, Database: { acquireCount: { r: 8807 } }, Collection: { acquireCount: { r: 8807 } } } protocol:op_query 75916ms
2017-11-24T09:24:06.006+0800 I COMMAND [conn3065049] command fs.files command: find { find: "files", filter: { operation.sha1Code: "6e267e339f19f537aca8d6388974a8c3429zbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:1 docsExamined:1 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8620 nreturned:1 reslen:651 locks:{ Global: { acquireCount: { r: 17242 }, acquireWaitCount: { r: 676 }, timeAcquiringMicros: { r: 11482889 } }, Database: { acquireCount: { r: 8621 } }, Collection: { acquireCount: { r: 8621 } } } protocol:op_query 48621ms
2017-11-24T09:24:06.014+0800 I COMMAND [conn3127183] command fs.files command: find { find: "files", filter: { operation.sha1Code: "2c06710b75d4b4c92c0cdffb5350659bb74zbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:0 docsExamined:0 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8412 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 16826 }, acquireWaitCount: { r: 510 }, timeAcquiringMicros: { r: 10260087 } }, Database: { acquireCount: { r: 8413 } }, Collection: { acquireCount: { r: 8413 } } } protocol:op_query 37187ms
2017-11-24T09:27:38.725+0800 I COMMAND [conn3032706] command fs.files command: find { find: "files", filter: { operation.sha1Code: "acca33867df96b97a05bdbd81f2aee13a50zbdba", operation.is_prefix: true, operation.des_url: "sd/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.des_url: 1.0 } keysExamined:103934 docsExamined:103934 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:4886 nreturned:1 reslen:719 locks:{ Global: { acquireCount: { r: 9774 }, acquireWaitCount: { r: 316 }, timeAcquiringMicros: { r: 471809 } }, Database: { acquireCount: { r: 4887 } }, Collection: { acquireCount: { r: 4887 } } } protocol:op_query 24573ms
2017-11-24T09:28:02.643+0800 I COMMAND [conn3062092] command fs.files command: find { find: "files", filter: { operation.sha1Code: "0a7dab951084d5a88a1ca736319998237fazbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:0 docsExamined:0 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8126 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 16254 }, acquireWaitCount: { r: 68 }, timeAcquiringMicros: { r: 819435 } }, Database: { acquireCount: { r: 8127 } }, Collection: { acquireCount: { r: 8127 } } } protocol:op_query 12645ms
```


但是在2017-11-24T09:28:02的时候，该SQL重新生成执行计划，然后重新选择operation.sha1Code: 1.0索引，更新cache中执行计划，然后后续类似SQL都走operation.sha1Code: 1.0索引，这样MongoDB又恢复了正常，相关日志如下：

```
19564934:2017-11-24T09:28:02.643+0800 I COMMAND [conn3062092] command fs.files command: find { find: "files", filter: { operation.sha1Code: "0a7dab951084d5a88a1ca736319998237fazbdba", operation.is_prefix: true, operation.des_url: "sinacn/" }, limit: 1, singleBatch: true } planSummary: IXSCAN { operation.sha1Code: 1.0 } keysExamined:0 docsExamined:0 fromMultiPlanner:1 replanned:1 cursorExhausted:1 keyUpdates:0 writeConflicts:0 numYields:8126 nreturned:0 reslen:114 locks:{ Global: { acquireCount: { r: 16254 }, acquireWaitCount: { r: 68 }, timeAcquiringMicros: { r: 819435 } }, Database: { acquireCount: { r: 8127 } }, Collection: { acquireCount: { r: 8127 } } } protocol:op_query 12645ms
```


那我们看看它为什么可以触发重新生成执行计划呢，而不采用cache中的执行计划。首先我们看最开始重新生成执行计划选择operation.des_url索引的SQL，operation.des_url字段索引的命中次数是103934，也就是说后续的SQL在判断是否采用cache中的执行计划时需要扫描103934*10次，如果没有超过101次Advanced状态（也是通过索引在collection中拿到满足条件的记录则返回Advanced累积次数），那么则放弃cache中的执行计划并且重新生成执行计划。我们看看上面这条SQL的operation.des_url: “sinacn/”字段索引命中记录数是多少？

```
comos_temp:SECONDARY> db.files.find({"operation.des_url":"sinacn/"}).count()
147195372
```


OK，我们再看这个SQL的nreturned是0，说明符合查询条件的记录是0，那及时全部扫描完成103934*10也达不到101次Advanced，别忘了我们还有一个条件，触发 IS_EOF就会直接采用cache中的执行计划，但是”operation.des_url”:”sinacn/”的记录数量远远大于103934*10，所以也不可能达到

`IS_EOF`

的状态。那么此时该SQL在扫描完103934*10次后就会触发重新生成执行计划。

这里是不是就明白为什么之前的故障自动恢复了，因为它出现了一个operation.des_url字段索引远大于cache执行计划中索引字段命中次数*10。而且也达不到101次advanced，所以只能重新生成执行计划。当然大家可以在看看2017年之前的日志，都是这种情况。

## 4、遇到该Bug如何处理

我们通过上述分析得出该次故障确实是由于MongoDB自身执行计划cache机制导致，那我们有什么应对方案呢？

1、重启实例

这个是最粗暴的方式，针对于MongoDB异常夯住不能登录的情况

2、清理cache中的执行计划

- 列出cache中保存的SQL过滤条件

```
db.files.getPlanCache().listQueryShapes()
comos_temp:SECONDARY> db.files.getPlanCache().listQueryShapes()
[
{
"query" : {
"operation.sha1Code" : "acca33867df96b97a05bdbd81f2aee13a50zbdba",
"operation.is_prefix" : true,
"operation.des_url" : "sh/"
},
"sort" : {
},
"projection" : {
}
}
]
```


- 根据该条件查看cache中的执行计划

```
db.files.getPlanCache().getPlansByQuery({"operation.sha1Code":"acca33867df96b97a05bdbd81f2aee13a50zbdba","operation.is_prefix":true,"operation.des_url":"sh/"})
```


- 清理该cache中的执行计划

```
db.files.getPlanCache().clear()
```


当然我们在紧急的情况下直接执行第三条命令即可。

3、反馈官方，优化执行计划cache机制，进行MongoDB升级

https://jira.mongodb.org/browse/SERVER-34785

## 5、附录

最后笔者贴上MongoDB优化器调用栈，方便撸源码的同学快速上手

```
mongo::IndexScan::work
mongo::FetchStage::work
mongo::CachedPlanStage::pickBestPlan
mongo::PlanExecutor::pickBestPlan
mongo::PlanExecutor::make
mongo::PlanExecutor::make
mongo::getExecutor
mongo::getExecutorFind
mongo::FindCmd::run
mongo::Command::run
mongo::Command::execCommand
mongo::runCommands
mongod.exe!mongo::`anonymous namespace'::receivedRpc
mongo::assembleResponse
mongo::`anonymous namespace'::MyMessageHandler::process
mongo::PortMessageServer::handleIncomingMsg
```


## 六、个人简介

赵景波，2017 DTCC 讲师，CRUG用户组成员，前新浪NoSQL服务负责人，热衷于开源DB内部原理探究。