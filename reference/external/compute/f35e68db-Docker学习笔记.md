---
title: Docker学习笔记
source: https://www.jianshu.com/p/5cced34a9483
kind: external
domain: compute
original_date: 2017-04-28
fetched_at: 2026-05-16
bookmark_title: Docker学习笔记 - 简书
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.jianshu.com](https://www.jianshu.com/p/5cced34a9483)
> 原始日期：2017-04-28
> 抓取日期：2026-05-16

# Docker学习笔记

原文地址：LoveDev

Docker相对于传统意义上的虚拟机最大的区别就在于传统虚拟机是虚拟出一套硬件后，再在上面运行一个完整的操作系统，再把需要运行的应用装在操作系统中运行。Docker在宿主的内核中运行应用进程，没有自己的内核，没有虚拟硬件，比起传统虚拟机更加轻快。

## Docker基本概念

- 镜像：操作系统
- 容器：容器是独立运行的一个或一组应用，以及它们的运行态环境，镜像和容器的关系就像是
**面向对象**中的**类**和**实例** - 仓库：镜像需要存储和分发，仓库用来存储镜像

### Docker Registry

一个Docker Registry中可以包含多个仓库

### Docker Hub

最常使用的 Registry 公开服务是官方的 Docker Hub，这也是默认的 Registry。

### Docker Registry 公开服务

国内访问Registry 公开服务会有些慢（原因你懂得），国内云服务商提供了针对 Docker Hub 的镜像服务（Registry Mirror），这些镜像服务被称为**加速器**，常见的有 阿里云加速器、DaoCloud 加速器、灵雀云加速器等。

配置如下：

国内也有一些云服务商提供类似于 Docker Hub 的公开服务。比如 时速云镜像仓库、网易云镜像服务、DaoCloud 镜像市场、阿里云镜像库等。

## 镜像

### 获取镜像

```
$ docker pull [选项][Docker Registry地址]<仓库名>:<标签>
例如：$ docker pull ubuntu
```


- Docker Registry地址：地址的格式一般是 <域名/IP>[:端口号]

，默认地址是 Docker Hub。 - 仓库名：如之前所说，这里的仓库名是两段式名称，既 <用户名>/<软件名>

。对于 Docker Hub，如果不给出用户名，则默认为 library

，也就是官方镜像。

### 运行镜像

有个镜像就可以以这个镜像运行一个容器，以上面ubuntu为例运行一个容器。

```
$ docker run -it --rm ubuntu bash
```


-
`docker run`

：就是运行容器的命令 -
`-it`

：其实是两个参数，`-i`

：交互式操作，`-t`

：终端 -
`--rm`

：容器退出后随之将其删除 -
`ubuntu`

：用 ubuntu 镜像为基础来启动容器 -
`bash`

：放在镜像名后的是**命令**，这里我们希望有个交互式 Shell，因此用的是 bash

```
$ docker run -d -p 22 -p 80:8080 ubuntu/kevin /usr/sbin/sshd -D
```


-
`-d`

：容器后台运行 -
`-p`

：指定端口设置 -
`-p 80:8080`

：端口映射，省略80表示把容器端口8080映射到一个动态端口 -
`/usr/sbin/sshd`

：启动 ssh 服务 -
`-D`

：容器长时间运行

*注：*`exit`

：退出容器

### 列出镜像

列出下载的镜像用`docker images`

命令。

列表中的镜像体积综合并非实际硬盘消耗，由于 Docker 镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。

#### 虚悬镜像

镜像既没有仓库名，也没有标签，均为 `<none>`

。此类镜像为**虚悬镜像(dangling image)** ，下面命令专门显示此类镜像

```
$ docker images -f dangling=true
```


这类镜像已经失去了存在的价值，可以随意删除，删除命令如下

```
$ docker rmi $(docker images -q -f dangling=true)
```


#### 中间层镜像

为了加速镜像构建、重复利用资源，Docker 会利用 **中间层镜像**。所以在使用一段时间后，可能会看到一些依赖的中间层镜像。默认的 `docker images`

列表中只会显示顶层镜像，如果希望显示包括中间层镜像在内的所有镜像的话，需要加 `-a`

参数。

```
$ docker images -a
```


#### 列出部分镜像

根据仓库名列出镜像

```
$ docker image ubuntu
```


根据仓库名和标签

```
$ docker images ubunut:16.04
```


除此以外，`docker images`

还支持强大的过滤器参数 `--filter`

，或者简写 `-f`

。希望看到在`nginx`

之后建立的镜像，可以用下面的命令

```
$ docker images -f since=nginx
```


希望看到`nginx`

之前建立的镜像，`since`

换成`before`


### 保存镜像

可用下面命令保存镜像

```
$ docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
例如：docker commit -m "commit message" CONTAINER kevinlovedev/tomcat
```


-
`-m`

：保存commit信息

### 删除镜像

可用下面命令删除镜像

```
$ docker rmi [OPTIONS] IMAGE [IMAGE...]
```


## 容器

### 启动

所需主要命令为`docker run`


```
$ sudo docker run --name kevin -i -t ubuntu /bin/bash
```


-
`--name`

：为容器指定名称 -
`-i`

：保证容器中`STDIN`

是开启的 -
`-t`

：为创建的容器分配一个伪tty终端，这样容器才能提供一个交互式shell

`docker start`

命令直接将一个已经终止的容器启动运行

### 后台运行

更多的时候，需要让 Docker在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过添加 `-d`

参数来实现。

获取容器的输出信息，可以通过 `docker logs`

命令。

### 启动并进入

大多数情况，我们需要启动并且直接进入到容器里面。

```
$ docker run -it 容器名 bash
```


### 终止

可用 `docker stop [OPTIONS] CONTAINER [CONTAINER...]`

来终止一个运行中的容器。

*注：*此命令后面是`CONTAINER ID`

或者`NAME`

参数，可用`docker ps`

查看

此外，当Docker容器中指定的应用终结时，容器也自动终止。 例如对于上一章节中只启动了一个终端的容器，用户通过 `exit`

命令或 `Ctrl+d`

来退出终端时，所创建的容器立刻终止。

终止状态的容器可以用 `docker ps -a`

命令看到。

```
$ docker rm $(docker ps -qa --no-trunc --filter "status=exited") # 删除所有已退出容器
```


### 进入容器

在使用 `-d`

参数时，容器启动后会进入后台。 某些时候需要进入容器进行操作

可用`docker attach [OPTIONS] CONTAINER`

命令进入

### 导出容器

可用 `docker export`

命令。

```
$ docker export ubuntu:kevin > latest.tar
```


### 导入容器

可用 `docker improt`

命令。

*注：*用户既可以使用 `docker load`

来导入镜像存储文件到本地镜像库，也可以使用 `docker import`

来导入一个容器快照到本地镜像库。这两者的区别在于容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。

### 删除容器

可用 `docker rm`

来删除一个处于终止状态的容器

用 `docker ps -a`

命令可以查看所有已经创建的包括终止状态的容器，如果数量太多要一个个删除可能会很麻烦，用 `docker rm $(docker ps -a -q)`

可以全部清理掉。

## 仓库

仓库（Repository）是集中存放镜像的地方。

### 搜索

用户无需登录即可通过 `docker search`

命令来查找官方仓库中的镜像，并利用 `docker pull`

命令来将它下载到本地。

另外，查找的时候通过 `-s N`

参数可以指定仅显示评价为 `N`

星以上的镜像（已经过时，不推荐使用），最新版本使用

`--filter`

过滤查找。

利用下面命令下载到本地

```
$ sudo docker pull centos
```


### 数据管理

容器中管理数据主要有两种方式：

- 数据卷（Data volumes）
- 数据卷容器（Data volume containers）

#### 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

- 数据卷可以在容器之间共享和重用
- 对数据卷的修改会立马生效
- 对数据卷的更新，不会影响镜像
- 数据卷默认会一直存在，即使容器被删除

*注意：*数据卷的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的数据卷。

#### 容器和主机之间拷贝数据

拷贝容器文件到主机

```
$ docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH
例如：docker cp CONTAINER:/usr/local/tomcat/webapps/ROOT/index.html index.html
```


拷贝主机文件到容器

```
$ docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
例如：docker cp index.html CONTAINER:/usr/local/tomcat/webapps/ROOT/index.html
```


## Docker Machine

Docker Machine 是一个 docker 管理工具，主要解决两个问题：

- docker 只能运行在 Linux 上
- docker 只能管理运行本机的 docker 镜像

由于之前配置了 使用Docker Machine管理阿里云ECS ，Docker Machine使用一直不成功，报错信息如下：

```
Error setting machine configuration from flags provided: --engine-install-url cannot be used with the virtualbox driver, use --virtualbox-boot2docker-url instead
```


原因在于配置了阿里的一些参数，其中包括了`MACHINE_DOCKER_INSTALL_URL`

，这个是报错的主要原因，查了半天的资料终于爬出坑了

## Push本地镜像到DockerHub

首先镜像名称格式需要是`DockerHubName/RepositoryName`

，例如`kevinlovedev/tomcat`

，可用`docker commit`

命令保存镜像，`/`

前面是DockerHub昵称，`/`

后面是DockerHub仓库名。

保存镜像成功之后，需要用`docker login`

命令登陆到DockerHub。

最后用`docker push`

命令提交本地镜像到DockerHub仓库。

## Dockerfile

MAINTAINER：设置该镜像的作者。语法如下：

```
MAINTAINER kevin "kevinlovemail@gmail.com"
```


### build

使用了 `docker build`

命令进行镜像构建。其格式为：

```
$ docker build -t ubuntu:kevin .
```


`-t`

指定了最终镜像的名称 `ubuntu:kevin`


## 常见问题

### 无法删除一

```
Error response from daemon: conflict: unable to delete e4b9e4f71238 (must be forced) - image is being used by stopped container 1e359ad4363d
```


该容器是终止状态，需要将此容器从终止状态删除，然后再删除镜像

```
$ docker rm 1e359ad4363d # 删除终止容器
$ docker rmi e4b9e4f71238 # 删除镜像
```


### 无法删除二

```
Error response from daemon: conflict: unable to delete 1dc4f730b414 (cannot be forced) - image has dependent child images
```


先删除依赖，如果 `IMAGE ID`

相同的话，根据 `TAG`

删除

```
$ docker rm REPOSITORY:TAG # 根据TAG删除容器
```


### 语法错误

```
Error response from daemon: Unknown instruction: RUNCMD
```


### can't initialize iptables

```
can't initialize iptables table `filter': Permission denied (you must be root)
```


启动容器时加入参数 `--privileged=true`