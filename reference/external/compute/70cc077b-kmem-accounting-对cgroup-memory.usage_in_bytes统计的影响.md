---
title: kmem accounting 对cgroup memory.usage_in_bytes统计的影响-腾讯云开发者社区-腾讯云
source: https://cloud.tencent.com/developer/article/1637693
kind: external
domain: compute
original_date: 2020-06-02
fetched_at: 2026-05-16
bookmark_title: kmem accounting 对cgroup memory.usage_in_bytes统计的影响 - 云+社区 - 腾讯云
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[cloud.tencent.com](https://cloud.tencent.com/developer/article/1637693)
> 原始日期：2020-06-02
> 抓取日期：2026-05-16

# kmem accounting 对cgroup memory.usage_in_bytes统计的影响-腾讯云开发者社区-腾讯云

cdh

修改于 2020-06-02 23:31:11

修改于 2020-06-02 23:31:11

**1. 首先看下内核如何设置kmem accounting?**

当往memory cgroup所在的目录下文件memory.kmem.limit_in_bytes写入值时，内核会调用mem_cgroup_write:

写入memory.kmem.limit_in_bytes的值范围为0 至 -1，在内核因为采用了页对齐，所以实际以4096倍数增长，最大值为RESOURCE_MAX。写入-1时转化为最大值RESOURCE_MAX(9223372036854775807)

```
static int memcg_update_kmem_limit(struct cgroup *cont, unsigned long limit)
{
int ret = -EINVAL;
#ifdef CONFIG_MEMCG_KMEM
struct mem_cgroup *memcg = mem_cgroup_from_cont(cont);
/*
* For simplicity, we won't allow this to be disabled. It also can't
* be changed if the cgroup has children already, or if tasks had
* already joined.
*
* If tasks join before we set the limit, a person looking at
* kmem.usage_in_bytes will have no way to determine when it took
* place, which makes the value quite meaningless.
*
* After it first became limited, changes in the value of the limit are
* of course permitted.
*/
mutex_lock(&memcg_create_mutex);
mutex_lock(&memcg_limit_mutex);
//未使能kmem accounting且用户输入值不为-1
if (!memcg->kmem_account_flags && limit != PAGE_COUNTER_MAX) {
if (cgroup_task_count(cont) || memcg_has_children(memcg)) {
ret = -EBUSY;
goto out;
}
ret = page_counter_limit(&memcg->kmem, limit);
VM_BUG_ON(ret);
ret = memcg_update_cache_sizes(memcg);
if (ret) {
page_counter_limit(&memcg->kmem, PAGE_COUNTER_MAX);
goto out;
}
static_key_slow_inc(&memcg_kmem_enabled_key);
/*
* setting the active bit after the inc will guarantee no one
* starts accounting before all call sites are patched
*/
memcg_kmem_set_active(memcg);//设置kmem_accout_flags代表已使能kmem accounting
/*
* kmem charges can outlive the cgroup. In the case of slab
* pages, for instance, a page contain objects from various
* processes, so it is unfeasible to migrate them away. We
* need to reference count the memcg because of that.
*/
mem_cgroup_get(memcg);
} else
ret = page_counter_limit(&memcg->kmem, limit);
out:
mutex_unlock(&memcg_limit_mutex);
mutex_unlock(&memcg_create_mutex);
#endif
return ret;
}
```


设置kmem_account_flags表示开启kmem accounting功能

2. 如何使用kmem accounting

Centos默认开启了kmem accounting，往对应的memory cgroup目录下文件memory.kmem.limit_in_bytes写入对应的值可以

限制cgroup kmem大小：

如果既要开启kmem accounting又不想对所在的memory cgroup的kmem进行限制，可按如下方式设置：

先往memory.kmem.limit_in_bytes写入一个不为RESOURCE_MAX的任意非负数值开启kmem accounting功能

echo 0 > memory.kmem.limit_in_bytes

再对memory.kmem.limit_in_bytes写入-1值，内核将-1值转为RESOURCE_MAX写入memory.kmem.limit_in_bytes

echo -1 > memory.kmem.limit_in_bytes

**3.内核如何统计用户读取到的memory.usage_in_bytes值？**

用户态读取memory cgroup的memory.usage_in_bytes值时，在内核中实际上读取的是memcg->res (未使能swap)

未使能kmem accounting功能，则内核使用的内存（share memory和file cache除外）比如slab不会被计算到memory.usage_in_bytes中

__mem_cgroup_try_charge

**4. 测试程序验证kmem accounting对memory.usage_in_bytes影响：**

测试代码查看附件，测试方法是通过建立20个容器，每个容器建立2000个tcp连接，通过建立大量的TCP连接触发内核分配使用slab内存：

4.1 Disable kmem accounting:

<1> 创建20个容器，每个容器建立20000个连接：

./setup.sh 20 20000 127.0.0.1

<2> 等待一段时间待所有容器完成连接建立，内存统计稳定下来后查看20个容器所在的父memory cgroup test-docker-memory的内存统计memory.usage_in_bytes与total_rss非常接近：

<3> 对比执行测试用例前后node的/proc/meminfo信息：

added（anon）= （4831280+21848）-（374368+18484）=4853128 - 392852 = 4460276 KB

added （slab）= 4120476 - 132064 = 3988412 KB

从测试数据可以看到内核SLAB占用的内存没有被计算到memory.usage_in_bytes中

4.2 Enable kmem accounting:

<1> 创建20个容器，每个容器建立20000个连接：

./setup.sh 20 20000 127.0.0.1

<2> 等待一段时间待所有容器完成连接建立，内存统计稳定下来后查看20个容器所在的父memory cgroup test-docker-memory的内存统计memory.usage_in_bytes比total_rss多出约3G内存

<3> 对比执行测试用例前后node的/proc/meminfo信息：

added （anon）= （4929264+21872）-（325160+18468）=4951136- 343628= 4607508 KB

added （slab）= 4306076- 141828= 4164248 KB = 4G

从测试数据可以看到开启kmem accounting后内核SLAB占用的内存也被计算到memory.usage_in_bytes中。

注：运行测试用例前后对比/proc/meminfo中增加的slab内存包含了node自身占用的内存，不仅仅是测试容器所在的memory cgroup test-docker-memory.

附上测试工具源码：

#cat readme

go build server.go

go build client.go

#启动20个容器，每个容器建立20000个连接

./setup.sh 20 20000 127.0.0.1

#cat server.go

package main

import (

"io/ioutil"

"log"

// "net/http"

"net"

"io"

)

func main() {

ln, err := net.Listen("tcp", ":8972")

if err != nil {

panic(err)

}

/* go func() {

if err := http.ListenAndServe(":6060", nil); err != nil {

log.Fatalf("pprof failed: %v", err)

}

}()

*/

var connections []net.Conn

defer func() {

for _, conn := range connections {

conn.Close()

}

}()

for {

conn, e := ln.Accept()

if e != nil {

if ne, ok := e.(net.Error); ok && ne.Temporary() {

log.Printf("accept temp err: %v", ne)

continue

}

log.Printf("accept err: %v", e)

return

}

go handleConn(conn)

connections = append(connections, conn)

if len(connections)%100 == 0 {

log.Printf("total number of connections: %v", len(connections))

}

}

}

func handleConn(conn net.Conn) {

io.Copy(ioutil.Discard, conn)

}

#cat client.go

package main

import (

"log"

"net"

"fmt"

"flag"

"time"

)

var (

ip = flag.String("ip", "127.0.0.1", "server IP")

connections = flag.Int("conn", 1, "number of tcp connections")

)

func main() {

flag.Parse()

addr := *ip + ":8972"

log.Printf("连接到 %s", addr)

var conns []net.Conn

for i := 0; i < *connections; i++ {

c, err := net.DialTimeout("tcp", addr, 10*time.Second)

if err != nil {

fmt.Println("failed to connect", i, err)

i--

continue

}

conns = append(conns, c)

time.Sleep(time.Millisecond)

}

defer func() {

for _, c := range conns {

c.Close()

}

}()

log.Printf("完成初始化 %d 连接", len(conns))

/* tts := time.Second

if *connections > 100 {

tts = time.Millisecond * 5

}

for {

for i := 0; i < len(conns); i++ {

time.Sleep(tts)

conn := conns[i]

conn.Write([]byte("hello world\r\n"))

}

}

*/

for {

time.Sleep(3600)

}

}

#cat setup.sh

REPLICAS=$1

CONNECTIONS=$2

IP=$3

cgcreate -g memory:test-docker-memory

echo 0 > /sys/fs/cgroup/memory/test-docker-memory/memory.kmem.limit_in_bytes

echo -1 > /sys/fs/cgroup/memory/test-docker-memory/memory.kmem.limit_in_bytes

#go build --tags "static netgo" -o client client.go

for (( c=0; c<${REPLICAS}; c++ ))

do

# docker run -v $(pwd)/client:/client --name 1mclient_$c -d alpine /client \

# -conn=${CONNECTIONS} -ip=${IP}

docker run --cgroup-parent=/test-docker-memory --net=none -v /root/test:/test -idt --name test_$c --privileged csighub.tencentyun.com/admin/tlinux2.2-bridge-tcloud-underlay:latest /test/test.sh

done

#cat test.sh

#!/bin/bash

/test/server &

/test/client -conn=100000

原创声明：本文系作者授权腾讯云开发者社区发表，未经许可，不得转载。

如有侵权，请联系 cloudcommunity@tencent.com 删除。

原创声明：本文系作者授权腾讯云开发者社区发表，未经许可，不得转载。

如有侵权，请联系 cloudcommunity@tencent.com 删除。

评论

登录后参与评论

推荐阅读