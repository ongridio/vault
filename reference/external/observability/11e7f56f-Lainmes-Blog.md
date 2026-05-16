---
title: Lainme's Blog
source: https://www.lainme.com/doku.php/blog/2011/01/%E9%80%8F%E8%BF%87%E4%BB%A3%E7%90%86%E8%BF%9E%E6%8E%A5ssh
kind: external
domain: observability
original_date: 2023-03-09
fetched_at: 2026-05-16
bookmark_title: 透过代理连接SSH [Lainme's Blog]
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.lainme.com](https://www.lainme.com/doku.php/blog/2011/01/%E9%80%8F%E8%BF%87%E4%BB%A3%E7%90%86%E8%BF%9E%E6%8E%A5ssh)
> 原始日期：2023-03-09
> 抓取日期：2026-05-16

# Lainme's Blog

# 透过代理连接SSH

虽然折腾PcmanFM没什么成效，却终于知道如何让SSH通过代理了。这么一来，使用GitHub和Launchpad都方便了不少。

这是通过SSH的ProxyCommand来完成的。可以用

man ssh_config

来查看相关信息。

## 通过SSH代理(SSH over SSH)

使用nc命令(netcat)实现，假设本地SSH代理的监听端口是3000，则ProxyCommand为

ProxyCommand nc -x 127.0.0.1:3000 %h %p

其中%h表示目标地址，%p是目标端口。这句可以用在命令行里，例如

ssh -oProxyCommand="nc -x 127.0.0.1:3000 %h %p" git@github.com

或者写入config文件(参见使用SSH CONFIG)

Host 名称 HostName 域名/IP User 用户 ProxyCommand nc -x 127.0.0.1:3000 %h %p

nc也可以用于HTTPS代理，这需要指定所使用的协议，即添加 -X connect 参数。比如ssh_config中的例子

ProxyCommand /usr/bin/nc -X connect -x 192.0.2.0:8080 %h %p

netcat也有很多其他用途，有兴趣可以看看

## 通过HTTP代理(SSH over HTTP)

需要corkscrew这个软件

sudo aptitude install corkscrew

基本的语句是

ProxyCommand corkscrew 代理服务器地址 端口 %h %p

如果HTTP代理需要用户名/密码验证,则需要写上代理验证文件。假设代理服务器是192.168.0.1:808。用户名密码是name:pass，打算存放在~/.ssh/proxyauth。则有

ProxyCommand corkscrew 192.168.0.1 808 %h %p ~/.ssh/proxyauth

新建~/.ssh/proxyauth文件，写上

name:pass

## 为dput设置代理(PPA上传)

很多时候连接到Launchpad的速度是非常慢的，找个好的代理可以改善这一情况。下载给apt设置代理就行了，方法多样。上传就需要让dput能通过代理，而它似乎没有内建的代理支持？不过dput支持sftp上传，也就可以使用给SSH设代理的方式来进行。

要使用sftp上传方式，先要生成相应的SSH key，在终端下执行

ssh-keygen -t rsa

全部默认按回车，这里没有设置密码。

到launchpad的个人主页上去，找到“SSH keys:“，点击旁边的小图标进行编辑。将~/.ssh/id_rsa.pub的内容粘贴到文本框里，提交，这样就导入了公钥

在家目录下新建~/.dput.cf文件，内容如下(假设用户名是test)

[ppa] fqdn = ppa.launchpad.net method = sftp incoming = ~%(ppa)s/ubuntu login = test

编辑~/.ssh/config文件，添加如下

Host *.launchpad.net User test ProxyCommand (相应的代理命令，如上)

注意：要安装bzrtools包才能正常上传。。

最后：我果然还是不习惯截图啊