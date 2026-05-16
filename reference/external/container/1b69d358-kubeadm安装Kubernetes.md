---
title: kubeadm安装Kubernetes
source: https://juejin.im/post/5d4af431f265da03a653022a
kind: external
domain: container
author: Dlutzhangyi
original_date: 2019-08-07
fetched_at: 2026-05-16
bookmark_title: kubeadm安装Kubernetes - 掘金
tags: [external, container]
---

> [!info] 外部文章 · 自动导入
> 来源：[juejin.im](https://juejin.im/post/5d4af431f265da03a653022a)
> 作者：Dlutzhangyi
> 原始日期：2019-08-07
> 抓取日期：2026-05-16

# kubeadm安装Kubernetes

本次配置使用kubeadm安装Kubernetes，使用kubeadm init和kubeadm join两个命令可以很容易的初始化master节点和将node节点加入到master节点上。可关注我的csdn博客-blog.csdn.net/u010512429

# 环境准备

| 系统 | 版本 |
|---|---|
| Kubernetes | v1.15.1 |
| Docker | 18.06.1-ce |
| Centos7 | CentOS Linux release 7.1.1503 (Core) |

| 节点 | 主机名 | Roles |
|---|---|---|
| 10.32.170.109 | dx-ee-releng-webserver01 | master |
| 10.21.88.3 | gh-ee-plus09 | node1 |
| 10.4.95.37 | yf-ee-releng-plus02 | node2 |

# 基础配置

每个节点都需要进行如下配置，包括master和node。

## 安装Docker

使用yum安装docker

```
#使用yum安装docker-ce
yum install -y docker-ce-18.06.1.ce-3.el7
#创建docker daemon的配置文件
mkdir /etc/docker && touch /etc/docker/daemon.json
#配置国内镜像仓库加速器。
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://hdi5v8p1.mirror.aliyuncs.com"]
}
EOF
#启动docker
systemctl enable docker && service docker restart
```


## 关闭防火墙

```
systemctl stop firewalld
systemctl disable firewalld
```


## 关闭SELinux

```
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
setenforce 0
```


## 关闭swap

```
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab.hd
```


## 配置转发参数

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```


## 配置Kubernetes阿里云源

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```


## 安装Kubernetes相关组件

```
yum install kubelet kubeadm kubectl -y
systemctl enable kubelet && systemctl start kubelet
```


## 加载IPVS内核

加载ipvs内核，并添加到开机启动文件中

```
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
cat <<EOF >> /etc/rc.local
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
EOF
```


# 安装Master节点

## 执行kubeadm init 失败

在master节点上切换到root，并执行kubeadm init。发现failed to pull image。因为国内无法访问Google的镜像源，因此需要换另一种方式下载镜像源。这里我们采用的办法使用阿里云的镜像源下载镜像并对镜像重新打tag。

```
[root@dx-ee-releng-webserver01 docker]# kubeadm init --kubernetes-version=v1.15.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12
W0731 21:25:35.776310 6324 version.go:98] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0731 21:25:35.776456 6324 version.go:99] falling back to the local client version: v1.15.1
[init] Using Kubernetes version: v1.15.1
[preflight] Running pre-flight checks
[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
error execution phase preflight: [preflight] Some fatal errors occurred:
[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-apiserver:v1.15.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-controller-manager:v1.15.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-scheduler:v1.15.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-proxy:v1.15.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[ERROR ImagePull]: failed to pull image k8s.gcr.io/pause:3.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[ERROR ImagePull]: failed to pull image k8s.gcr.io/etcd:3.3.10: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[ERROR ImagePull]: failed to pull image k8s.gcr.io/coredns:1.3.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```


## 手动下载镜像

这里使用从阿里镜像源下载所需要的镜像，并对下载后的镜像重新打上tag。需要的具体镜像及对应的版本可以通过kubeadm config images list指令查看

```
[root@dx-ee-releng-webserver01 ~]# kubeadm config images list
W0731 21:29:32.916158 14480 version.go:98] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0731 21:29:32.916248 14480 version.go:99] falling back to the local client version: v1.15.1
k8s.gcr.io/kube-apiserver:v1.15.1
k8s.gcr.io/kube-controller-manager:v1.15.1
k8s.gcr.io/kube-scheduler:v1.15.1
k8s.gcr.io/kube-proxy:v1.15.1
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```


下面提供了一个下载镜像的脚本

```
#########################################################################
# File Name: pull_master_image.sh
# Description: pull_master_image.sh
# Author: zhangyi
# mail: 450575982@qq.com
# Created Time: 2019-07-31 21:38:14
#########################################################################
#!/bin/bash
kube_version=:v1.15.1
kube_images=(kube-proxy kube-scheduler kube-controller-manager kube-apiserver)
addon_images=(etcd-amd64:3.3.10 coredns:1.3.1 pause-amd64:3.1)
for imageName in ${kube_images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName-amd64$kube_version
docker image tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName-amd64$kube_version k8s.gcr.io/$imageName$kube_version
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName-amd64$kube_version
done
for imageName in ${addon_images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
docker image tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
docker tag k8s.gcr.io/etcd-amd64:3.3.10 k8s.gcr.io/etcd:3.3.10
docker image rm k8s.gcr.io/etcd-amd64:3.3.10
docker tag k8s.gcr.io/pause-amd64:3.1 k8s.gcr.io/pause:3.1
docker image rm k8s.gcr.io/pause-amd64:3.1
```


成功执行后执行docker images指令，可以发现需要的镜像已经被下载了

```
[root@dx-ee-releng-webserver01 ~]# docker images
REPOSITORY TAG IMAGE ID CREATED SIZE
k8s.gcr.io/kube-scheduler v1.15.1 b0b3c4c404da 2 weeks ago 81.1MB
k8s.gcr.io/kube-controller-manager v1.15.1 d75082f1d121 2 weeks ago 159MB
k8s.gcr.io/kube-apiserver v1.15.1 68c3eb07bfc3 2 weeks ago 207MB
k8s.gcr.io/kube-proxy v1.15.1 89a062da739d 2 weeks ago 82.4MB
k8s.gcr.io/coredns 1.3.1 eb516548c180 6 months ago 40.3MB
k8s.gcr.io/etcd 3.3.10 2c4adeb21b4f 8 months ago 258MB
k8s.gcr.io/pause 3.1 da86e6ba6ca1 19 months ago 742kB
```


## 执行kubeadm init

在成功下载镜像后，在master节点上执行如下kubeadm init --kubernetes-version=v1.15.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12指令用于初始化master节点。

```
[root@dx-ee-releng-webserver01 k8s]# kubeadm init --kubernetes-version=v1.15.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12
[init] Using Kubernetes version: v1.15.1
[preflight] Running pre-flight checks
[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [dx-ee-releng-webserver01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.32.170.109]
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [dx-ee-releng-webserver01 localhost] and IPs [10.32.170.109 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [dx-ee-releng-webserver01 localhost] and IPs [10.32.170.109 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 37.502506 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node dx-ee-releng-webserver01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node dx-ee-releng-webserver01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 0gpfb4.ee1nalr0vmzxa9cv
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 10.32.170.109:6443 --token 0gpfb4.ee1nalr0vmzxa9cv \
--discovery-token-ca-cert-hash sha256:f7abecbc3c8ff1ab9e91ff601cb16f952f90dfccd2d2ad71ed86c29e7363d699
```


如上所示，在初始化成功后，执行下面的命令

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## 配置网络

在kubeadm init成功初始化后，需要在master上配置网络，否则master节点上的coredns容器无法启动。

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```


# 安装Node节点

## 下载镜像

同样的，在Node节点上事先下载好需要的镜像，下面给了一份脚本用于下载Node节点需要的镜像。

```
#########################################################################
# File Name: pull_node_image.sh
# Description: pull_node_image.sh
# Author: zhangyi
# mail: 450575982@qq.com
# Created Time: 2019-08-06 10:45:43
#########################################################################
#!/bin/bash
kube_version=:v1.15.1
coredns_version=1.3.1
pause_version=3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64$kube_version
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64$kube_version k8s.gcr.io/kube-proxy$kube_version
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64$kube_version
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:$pause_version
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:$pause_version k8s.gcr.io/pause:$pause_version
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:$pause_version
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$coredns_version
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$coredns_version k8s.gcr.io/coredns:$coredns_version
docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$coredns_version
```


## 添加Node节点

使用kubeadm join将该node节点加入master节点

```
kubeadm join 10.32.170.109:6443 --token 0gpfb4.ee1nalr0vmzxa9cv \
--discovery-token-ca-cert-hash sha256:f7abecbc3c8ff1ab9e91ff601cb16f952f90dfccd2d2ad71ed86c29e7363d699
```


上述命令成功执行后，在master节点上执行kubectl get nodes，可以发现两个node节点和master节点都处于Ready的状态了。

```
[root@dx-ee-releng-webserver01 ~]# kubectl get nodes
NAME STATUS ROLES AGE VERSION
dx-ee-releng-webserver01 Ready master 27h v1.15.1
gh-ee-plus09.gh.sankuai.com Ready <none> 24h v1.15.1
yf-ee-releng-plus02.mt Ready <none> 23h v1.15.1
```


# FAQ

在配置Kubernetes的过程中，踩了不少的坑，下面将这些坑和大家分享一下

## 执行kubeadm init显示kubelet not running

执行kubeadm init执行，显示kubelet并没有running或者处于unhealthy的状态。

```
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
Unfortunately, an error has occurred:
timed out waiting for the condition
This error is likely caused by:
- The kubelet is not running
- The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)
- No internet connection is available so the kubelet cannot pull or find the following control plane images:
- k8s.gcr.io/kube-apiserver-amd64:v1.11.2
- k8s.gcr.io/kube-controller-manager-amd64:v1.11.2
- k8s.gcr.io/kube-scheduler-amd64:v1.11.2
- k8s.gcr.io/etcd-amd64:3.2.18
- You can check or miligate this in beforehand with "kubeadm config images pull" to make sure the images
are downloaded locally and cached.
If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
- 'systemctl status kubelet'
- 'journalctl -xeu kubelet'
Additionally, a control plane component may have crashed or exited when started by the container runtime.
To troubleshoot, list all containers using your preferred container runtimes CLI, e.g. docker.
Here is one example how you may list all Kubernetes containers running in docker:
- 'docker ps -a | grep kube | grep -v pause'
Once you have found the failing container, you can inspect its logs with:
- 'docker logs CONTAINERID'
couldn't initialize a Kubernetes cluster
```


用journalctl -xeu kubelet的观察kubelet的日志，发现一条Fatal的日志，根据该信息进行了查询，发现使用k8s的版本太新导致的，具体错误原因可参考：github.com/kubernetes/…

```
Aug 06 14:52:09 dx-ee-releng-webserver01 kubelet[29231]: F0806 14:52:09.563107 29231 kubelet.go:1370] Failed to start ContainerManager failed to initialize top level QOS containers: failed to update top level Burstable QOS cgroup : failed to set supported cgroup subsystems for cgroup [kubepods burstable]: Failed to find subsystem mount for required subsystem: pids
```


因此修改kubelet的启动文件，在ExecStart上添加 --feature-gates SupportPodPidsLimit=false --feature-gates SupportNodePidsLimit=false，修改后执行systemctl daemon-reload && systemctl restart kubelet。至此，kubelet已经能成功启动。

```
[root@dx-ee-releng-webserver01 ~]# vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --feature-gates SupportPodPidsLimit=false --feature-gates SupportNodePidsLimit=false
```


## 获取不到pods

执行kubectl get pods报错

```
[root@dx-ee-releng-webserver01 ~]# kubectl get pods --all-namespaces
error: the server doesn't have a resource type "pods"
```


解决方案：删除.kube路径下的http-cache文件夹，重新执行命令，发现能获取到所有的pods了

```
[root@dx-ee-releng-webserver01 ~]# cd ~/.kube/
[root@dx-ee-releng-webserver01 .kube]# rm -rf http-cache/
[root@dx-ee-releng-webserver01 .kube]# kubectl get pods --all-namespaces
NAMESPACE NAME READY STATUS RESTARTS AGE
kube-system coredns-5c98db65d4-gxwh6 0/1 Pending 0 2m48s
kube-system coredns-5c98db65d4-mc2p4 0/1 Pending 0 2m48s
kube-system etcd-dx-ee-releng-webserver01 1/1 Running 0 2m1s
kube-system kube-apiserver-dx-ee-releng-webserver01 1/1 Running 0 108s
kube-system kube-controller-manager-dx-ee-releng-webserver01 1/1 Running 0 2m4s
kube-system kube-proxy-4jtm8 1/1 Running 0 2m48s
kube-system kube-scheduler-dx-ee-releng-webserver01 1/1 Running 0 103s
```


# 参考

参考wiki