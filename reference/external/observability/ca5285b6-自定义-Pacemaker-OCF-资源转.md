---
title: 自定义 Pacemaker OCF 资源（转）
source: http://blog.csdn.net/tantexian/article/details/50160159
kind: external
domain: observability
author: 成就一亿技术人
original_date: 2026-04-02
fetched_at: 2026-05-16
bookmark_title: 自定义 Pacemaker OCF 资源（转） - CSDN博客
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](http://blog.csdn.net/tantexian/article/details/50160159)
> 作者：成就一亿技术人
> 原始日期：2026-04-02
> 抓取日期：2026-05-16

# 自定义 Pacemaker OCF 资源（转）

Pacemaker / Corosync 是 Linux 下一组常用的高可用集群系统。Pacemaker 本身已经自带了很多常用应用的管理功能。但是如果要使用 Pacemaker 来管理自己实现的服务或是一些别的没现成的东西可用的服务时，就需要自己实现一个资源了。

Pacemaker 的资源主要有两类，即 LSB 和 OCF。其中 LSB 即 Linux 标准服务，通常就是 /etc/init.d 目录下那些脚本。Pacemaker 可以用这些脚本来启停服务。在 ```
crm
ra list lsb
```

中可以看到。另一类 OCF 实际上是对 LSB 服务的扩展，增加了一些高可用集群管理的功能如故障监控等和更多的元信息。可以通过 ```
crm
ra list ocf
```

看到当前支持的资源。要让 pacemaker 可以很好的对服务进行高可用保障就得实现一个 OCF 资源。

Pacemaker 自带的资源管理程序都在 /usr/lib/ocf/resource.d 下。其中的 heartbeat 目录中就包含了那些自带的常用服务。那些服务的脚本可以作为我们自己实现时候的参考。

每个 OCF 资源是一个可执行文件。通过命令行参数和环境变量来接受来自 Pacemaker 的输入。下面是一个简单的例子，创建了一个名叫 test 的资源。```
crm
ra meta test
```

的结果如下：

```
crm(live)# ra meta test
ocf:heartbeat:test
A test resource
Parameters (* denotes required, [] the default):
service_path* (string): the path of service script
This is a required parameter. The path of service script.
probe_url* (string): the url to detect the status of the service
This is a required parameter. The url to detect the status of the service.
Operations' defaults (advisory minimum):
start timeout=60s
stop timeout=30s
status timeout=30s
monitor_0 interval=10s timeout=30s
```


和其他的资源一样，这个 test 资源也会接受若干的参数，以及对 start、stop 等命令有超时默认值等。下面就是这个资源的 bash 实现代码。实际也可以使用任何语言来实现。

```
#!/bin/bash
# OCF parameters:
# OCF_RESKEY_service_path : service path
# OCF_RESKEY_probe_url : probe url
: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/resource.d/heartbeat}
. ${OCF_FUNCTIONS_DIR}/.ocf-shellfuncs
SERVICE_PATH="${OCF_RESKEY_service_path}"
PROBE_URL="${OCF_RESKEY_probe_url}"
COMMAND=$1
SERVICE_NAME=test
metadata_test() {
cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="test">
<version>1.0</version>
<longdesc lang="en">
A test resource
</longdesc>
<shortdesc lang="en">test</shortdesc>
<parameters>
<parameter name="service_path" required="1" unique="1">
<longdesc lang="en">
This is a required parameter. The path of service script.
</longdesc>
<shortdesc>the path of service script</shortdesc>
<content type="string" default=""/>
</parameter>
<parameter name="probe_url" required="1" unique="1">
<longdesc lang="en">
This is a required parameter. The url to detect the status of the service.
</longdesc>
<shortdesc>the url to detect the status of the service</shortdesc>
<content type="string" default=""/>
</parameter>
</parameters>
<actions>
<action name="start" timeout="60s" />
<action name="stop" timeout="30s" />
<action name="status" timeout="30s" />
<action name="monitor" depth="0" timeout="30s" interval="10s" />
<action name="meta-data" timeout="5s" />
<action name="validate-all" timeout="5s" />
</actions>
</resource-agent>
END
return $OCF_SUCCESS
}
monitor_test() {
return $OCF_SUCCESS
}
start_test() {
return $OCF_SUCCESS
}
stop_test() {
return $OCF_SUCCESS
}
status_test() {
return $OCF_SUCCESS
}
validate_all_test() {
ocf_log info "validate_all_test"
if [[ ! -x "$SERVICE_PATH" ]]; then
ocf_log err "service_path is not found"
exit $OCF_ERR_CONFIGURED
fi
if [[ ! -x "$PROBE_URL" ]]; then
ocf_log err "probe_url is required"
exit $OCF_ERR_CONFIGURED
fi
return $OCF_SUCCESS
}
run_func() {
local cmd="$1"
ocf_log debug "Enter $SERVICE_NAME $cmd"
$cmd
local status=$?
ocf_log debug "Leave $SERVICE_NAME $cmd $status"
exit $status
}
case "$COMMAND" in
meta-data)
run_func meta-data
;;
start)
run_func start
;;
stop)
run_func stop
;;
status)
run_func status
;;
monitor)
run_func monitor
;;
validate-all)
run_func vaidate-all
;;
*)
exit $OCF_ERR_ARGS
;;
esac
```


首先说说如何接受输入。同标准的 Linux 服务脚本一样，OCF 资源也会以命令名为参数被调用，只是多了几个命令而已。如通过```
/usr/lib/ocf/resource.d/heartbeat/test
start
```

来启动服务，```
/usr/lib/ocf/resource.d/heartbeat/test
monitor
```

来监控服务状态。根据执行的返回码来判断是否成功。具体的返回值含义如下：

```
OCF_SUCCESS=0
OCF_ERR_GENERIC=1
OCF_ERR_ARGS=2
OCF_ERR_UNIMPLEMENTED=3
OCF_ERR_PERM=4
OCF_ERR_INSTALLED=5
OCF_ERR_CONFIGURED=6
OCF_NOT_RUNNING=7
```


对于 start、stop 等命令，一定要在服务启动/停止完成后返回。要保证在返回后，通过 monitor 命令检测状态时要能够成功。如果在未完成启动就返回如直接将命令放后台执行，会导致通过 monitor 命令检测状态时，由于此时还未启动完成而失败，导致被认为是故障，从而导致重启或切换。所有命令执行成功时应当返回 OCF_SUCCESS ，即 0。出错时根据具体情况返回对应错误码。如 start 启动失败，monitor 监控到服务故障时应当返回 OCF_NOT_RUNNING 。在检查参数错误时应当返回 OCF_ERR_CONFIGURED。其他错误一般可以返回 OCF_ERR_GENERIC。

参数则是通过环境变量传递。如 test 资源定义的两个参数：service_path 和 probe_url 会分别通过环境变量 $OCF_RESKEY_service_path 和 $OCF_RESKEY_service_path 传递。

meta-data 命令会输出一段 xml。作为资源的元信息，```
crm
ra meta test
```

的结果也是由此而来的。参见代码中那一大段 xml。每个资源的说明，参数定义，命令定义都由这个 xml 说明。基本上参照例子就能明白。

定义参数的 parameter/content 支持的类型有 string、integer、boolean。default 属性可以定义参数的默认值。需要注意的是，定义资源未指定参数时，指定的默认值并不会出现在 OCF_RESKEY_<参数名> 变量中，而是会放在另外的变量 OCF_RESKEY_<参数名>_default 中。

参考资料：

- http://www.linux-ha.org/doc/dev-guides/ra-dev-guide.html
- http://www.linux-ha.org/wiki/LSB_Resource_Agents
- http://www.linux-ha.org/wiki/OCF_Resource_Agents
- http://refspecs.linux-foundation.org/LSB_3.2.0/LSB-Core-generic/LSB-Core-generic/iniscrptact.html
- http://www.opencf.org/