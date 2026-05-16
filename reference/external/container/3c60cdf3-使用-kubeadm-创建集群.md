---
title: 使用 kubeadm 创建集群
source: https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
kind: external
domain: container
original_date: 2026-04-21
fetched_at: 2026-05-16
bookmark_title: 使用 kubeadm 创建集群 | Kubernetes
tags: [external, container]
---

> [!info] 外部文章 · 自动导入
> 来源：[kubernetes.io](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
> 原始日期：2026-04-21
> 抓取日期：2026-05-16

# 使用 kubeadm 创建集群

使用 `kubeadm`

，你能创建一个符合最佳实践的最小化 Kubernetes 集群。
事实上，你可以使用 `kubeadm`

配置一个通过
Kubernetes 一致性测试的集群。
`kubeadm`

还支持其他集群生命周期功能，
例如启动引导令牌和集群升级。

`kubeadm`

工具很棒，如果你需要：

- 一个尝试 Kubernetes 的简单方法。
- 一个现有用户可以自动设置集群并测试其应用程序的途径。
- 其他具有更大范围的生态系统和/或安装工具中的构建模块。

你可以在各种机器上安装和使用 `kubeadm`

：笔记本电脑、一组云服务器、Raspberry Pi 等。
无论是部署到云还是本地，你都可以将 `kubeadm`

集成到 Ansible 或 Terraform 这类预配置系统中。

要遵循本指南，你需要：

- 一台或多台运行兼容 deb/rpm 的 Linux 操作系统的计算机；例如：Ubuntu 或 CentOS。
- 每台机器 2 GB 以上的内存，内存不足时应用会受限制。
- 用作控制平面节点的计算机上至少有 2 个 CPU。
- 集群中所有计算机之间具有完全的网络连接。你可以使用公共网络或专用网络。

你还需要使用可以在新集群中部署特定 Kubernetes 版本对应的 `kubeadm`

。

Kubernetes 版本及版本偏差策略适用于
`kubeadm`

以及整个 Kubernetes。
查阅该策略以了解支持哪些版本的 Kubernetes 和 `kubeadm`

。
该页面是为 Kubernetes v1.36 编写的。

`kubeadm`

工具的整体特性状态为正式发布（GA）。一些子特性仍在积极开发中。
随着工具的发展，创建集群的实现可能会略有变化，但总体实现应相当稳定。

根据定义，在 `kubeadm alpha`

下的所有命令均在 Alpha 级别上受支持。

- 安装单个控制平面的 Kubernetes 集群
- 在集群上安装 Pod 网络，以便你的 Pod 可以相互连通

在所有主机上安装容器运行时和 kubeadm。 详细说明和其他前提条件，请参见安装 kubeadm。

如果你已经安装了 kubeadm， 请查看升级 Linux 节点文档的前两步， 了解如何升级 kubeadm 的说明。

升级时，kubelet 每隔几秒钟重新启动一次， 在 crashloop 状态中等待 kubeadm 发布指令。crashloop 状态是正常现象。 初始化控制平面后，kubelet 将正常运行。

kubeadm 与其他 Kubernetes 组件类似，会尝试在与主机默认网关关联的网络接口上找到可用的 IP 地址。 这个 IP 地址随后用于由某组件执行的公告和/或监听。

要在 Linux 主机上获得此 IP 地址，你可以使用以下命令：

```
ip route show # 查找以 "default via" 开头的行
```


如果主机上存在两个或多个默认网关，Kubernetes 组件将尝试使用遇到的第一个具有合适全局单播 IP 地址的网关。 在做出这个选择时，网关的确切顺序可能因不同的操作系统和内核版本而有所差异。

Kubernetes 组件不接受自定义网络接口作为选项，因此必须将自定义 IP 地址作为标志传递给所有需要此自定义配置的组件实例。

如果主机没有默认网关，并且没有将自定义 IP 地址传递给 Kubernetes 组件，此组件可能会因错误而退出。

要为使用 `init`

或 `join`

创建的控制平面节点配置 API 服务器的公告地址，
你可以使用 `--apiserver-advertise-address`

标志。
最好在 kubeadm API中使用
`InitConfiguration.localAPIEndpoint`

和 `JoinConfiguration.controlPlane.localAPIEndpoint`

来设置此选项。

对于所有节点上的 kubelet，`--node-ip`

选项可以在 kubeadm 配置文件
（`InitConfiguration`

或 `JoinConfiguration`

）的 `.nodeRegistration.kubeletExtraArgs`

中设置。

有关双协议栈细节参见使用 kubeadm 支持双协议栈。

你分配给控制平面组件的 IP 地址将成为其 X.509 证书的使用者备用名称字段的一部分。 更改这些 IP 地址将需要签署新的证书并重启受影响的组件，以便反映证书文件中的变化。 有关此主题的更多细节参见手动续期证书。

Kubernetes 项目不推荐此方法（使用自定义 IP 地址配置所有组件实例）。
Kubernetes 维护者建议设置主机网络，使默认网关 IP 成为 Kubernetes 组件自动检测和使用的 IP。
对于 Linux 节点，你可以使用诸如 `ip route`

的命令来配置网络；
你的操作系统可能还提供更高级的网络管理工具。
如果节点的默认网关是公共 IP 地址，你应配置数据包过滤或其他保护节点和集群的安全措施。

这个步骤是可选的，只适用于你希望 `kubeadm init`

和 `kubeadm join`

不去下载存放在
`registry.k8s.io`

上的默认容器镜像的情况。

当你在离线的节点上创建一个集群的时候，kubeadm 有一些命令可以帮助你预拉取所需的镜像。 阅读离线运行 kubeadm 获取更多的详情。

kubeadm 允许你给所需要的镜像指定一个自定义的镜像仓库。 阅读使用自定义镜像获取更多的详情。

控制平面节点是运行控制平面组件的机器， 包括 etcd（集群数据库） 和 API 服务器 （命令行工具 kubectl 与之通信）。

- （推荐）如果计划将单个控制平面 kubeadm 集群升级成高可用，
你应该指定
`--control-plane-endpoint`

为所有控制平面节点设置共享端点。 端点可以是负载均衡器的 DNS 名称或 IP 地址。 - 选择一个 Pod 网络插件，并验证是否需要为
`kubeadm init`

传递参数。 根据你选择的第三方网络插件，你可能需要设置`--pod-network-cidr`

的值。 请参阅安装 Pod 网络附加组件。

- （可选）
`kubeadm`

试图通过使用已知的端点列表来检测容器运行时。 使用不同的容器运行时或在预配置的节点上安装了多个容器运行时，请为`kubeadm init`

指定`--cri-socket`

参数。 请参阅安装运行时。

要初始化控制平面节点，请运行：

```
kubeadm init <args>
```


`--apiserver-advertise-address`

可用于为控制平面节点的 API 服务器设置广播地址，
`--control-plane-endpoint`

可用于为所有控制平面节点设置共享端点。

`--control-plane-endpoint`

允许 IP 地址和可以映射到 IP 地址的 DNS 名称。
请与你的网络管理员联系，以评估有关此类映射的可能解决方案。

这是一个示例映射：

```
192.168.0.102 cluster-endpoint
```


其中 `192.168.0.102`

是此节点的 IP 地址，`cluster-endpoint`

是映射到该 IP 的自定义 DNS 名称。
这将允许你将 `--control-plane-endpoint=cluster-endpoint`

传递给 `kubeadm init`

，
并将相同的 DNS 名称传递给 `kubeadm join`

。稍后你可以修改 `cluster-endpoint`

以指向高可用性方案中的负载均衡器的地址。

kubeadm 不支持将没有 `--control-plane-endpoint`

参数的单个控制平面集群转换为高可用性集群。

有关 `kubeadm init`

参数的更多信息，请参见 kubeadm 参考指南。

要使用配置文件配置 `kubeadm init`

命令，
请参见带配置文件使用 kubeadm init。

要自定义控制平面组件，包括可选的对控制平面组件和 etcd 服务器的活动探针提供 IPv6 支持， 请参阅自定义参数。

要重新配置一个已经创建的集群， 请参见重新配置一个 kubeadm 集群。

要再次运行 `kubeadm init`

，你必须首先卸载集群。

如果将具有不同架构的节点加入集群， 请确保已部署的 DaemonSet 对这种体系结构具有容器镜像支持。

`kubeadm init`

首先运行一系列预检查以确保机器为运行 Kubernetes 准备就绪。
这些预检查会显示警告并在错误时退出。然后 `kubeadm init`

下载并安装集群控制平面组件。这可能会需要几分钟。完成之后你应该看到：

```
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
/docs/concepts/cluster-administration/addons/
You can now join any number of machines by running the following on each node
as root:
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```


要使非 root 用户可以运行 kubectl，请运行以下命令，
它们也是 `kubeadm init`

输出的一部分：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


或者，如果你是 `root`

用户，则可以运行：

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```


`kubeadm init`

生成的 kubeconfig 文件 `admin.conf`

包含一个带有 `Subject: O = kubeadm:cluster-admins, CN = kubernetes-admin`

的证书。
`kubeadm:cluster-admins`

组被绑定到内置的 `cluster-admin`

ClusterRole 上。
不要与任何人共享 `admin.conf`

文件。

`kubeadm init`

生成另一个 kubeconfig 文件 `super-admin.conf`

，
其中包含带有 `Subject: O = system:masters, CN = kubernetes-super-admin`

的证书。
`system:masters`

是一个紧急访问、超级用户组，可以绕过授权层（例如 RBAC）。
不要与任何人共享 `super-admin.conf`

文件，建议将其移动到安全位置。

有关如何使用 `kubeadm kubeconfig user`

为其他用户生成 kubeconfig
文件，请参阅为其他用户生成 kubeconfig 文件。

记录 `kubeadm init`

输出的 `kubeadm join`

命令。
你需要此命令将节点加入集群。

令牌用于控制平面节点和加入节点之间的相互身份验证。
这里包含的令牌是密钥。确保它的安全，
因为拥有此令牌的任何人都可以将经过身份验证的节点添加到你的集群中。
可以使用 `kubeadm token`

命令列出，创建和删除这些令牌。
请参阅 kubeadm 参考指南。

本节包含有关网络设置和部署顺序的重要信息。在继续之前，请仔细阅读所有建议。

**你必须部署一个基于 Pod 网络插件的容器网络接口（CNI），
以便你的 Pod 可以相互通信。在安装网络之前，集群 DNS (CoreDNS) 将不会启动。**

- 注意你的 Pod 网络不得与任何主机网络重叠：如果有重叠，你很可能会遇到问题。
（如果你发现网络插件的首选 Pod 网络与某些主机网络之间存在冲突，
则应考虑使用一个合适的 CIDR 块来代替，
然后在执行
`kubeadm init`

时使用`--pod-network-cidr`

参数并在你的网络插件的 YAML 中替换它）。

- 默认情况下，
`kubeadm`

将集群设置为使用和强制使用 RBAC（基于角色的访问控制）。 确保你的 Pod 网络插件支持 RBAC，以及用于部署它的清单也是如此。

- 如果要为集群使用 IPv6（双协议栈或仅单协议栈 IPv6 网络）， 请确保你的 Pod 网络插件支持 IPv6。 IPv6 支持已在 CNI v0.6.0 版本中添加。

kubeadm 应该是与 CNI 无关的，对 CNI 驱动进行验证目前不在我们的端到端测试范畴之内。 如果你发现与 CNI 插件相关的问题，应在其各自的问题跟踪器中记录而不是在 kubeadm 或 kubernetes 问题跟踪器中记录。

一些外部项目为 Kubernetes 提供使用 CNI 的 Pod 网络， 其中一些还支持网络策略。

请参阅实现 Kubernetes 网络模型的附加组件列表。

请参阅安装插件页面， 了解 Kubernetes 支持的网络插件的非详尽列表。 你可以使用以下命令在控制平面节点或具有 kubeconfig 凭据的节点上安装 Pod 网络附加组件：

```
kubectl apply -f <add-on.yaml>
```


只有少数 CNI 插件支持 Windows， 更多详细信息和设置说明请参阅添加 Windows 工作节点。

每个集群只能安装一个 Pod 网络。

安装 Pod 网络后，你可以通过在 `kubectl get pods --all-namespaces`

输出中检查
CoreDNS Pod 是否 `Running`

来确认其是否正常运行。
一旦 CoreDNS Pod 启用并运行，你就可以继续加入节点。

如果你的网络无法正常工作或 CoreDNS 不在 `Running`

状态，请查看 `kubeadm`

的故障排除指南。

默认情况下，kubeadm 启用
NodeRestriction
准入控制器来限制 kubelet 在节点注册时可以应用哪些标签。准入控制器文档描述 kubelet `--node-labels`

选项允许使用哪些标签。
其中 `node-role.kubernetes.io/control-plane`

标签就是这样一个受限制的标签，
kubeadm 在节点创建后使用特权客户端手动应用此标签。
你可以使用一个有特权的 kubeconfig，比如由 kubeadm 管理的 `/etc/kubernetes/admin.conf`

，
通过执行 `kubectl label`

来手动完成操作。

默认情况下，出于安全原因，你的集群不会在控制平面节点上调度 Pod。 如果你希望能够在单机 Kubernetes 集群等控制平面节点上调度 Pod，请运行：

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```


输出看起来像：

```
node "test-01" untainted
...
```


这将从任何拥有 `node-role.kubernetes.io/control-plane:NoSchedule`

污点的节点（包括控制平面节点）上移除该污点。
这意味着调度程序将能够在任何地方调度 Pod。

此外，你可以执行以下命令从控制平面节点中删除
`node.kubernetes.io/exclude-from-external-load-balancers`

标签，这会将其从后端服务器列表中排除：

```
kubectl label nodes --all node.kubernetes.io/exclude-from-external-load-balancers-
```


请参阅使用 kubeadm 创建高可用性集群， 了解通过添加更多控制平面节点创建高可用性 kubeadm 集群的步骤。

工作节点是工作负载运行的地方。

以下页面展示如何使用 `kubeadm join`

命令将 Linux 和 Windows 工作节点添加到集群：

为了使 kubectl 在其他计算机（例如笔记本电脑）上与你的集群通信， 你需要将管理员 kubeconfig 文件从控制平面节点复制到工作站，如下所示：

```
scp root@<control-plane-host>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
```


上面的示例假定为 root 用户启用了 SSH 访问。如果不是这种情况，
你可以使用 `scp`

将 `admin.conf`

文件复制给其他允许访问的用户。

admin.conf 文件为用户提供了对集群的超级用户特权。
该文件应谨慎使用。对于普通用户，建议生成一个你为其授予特权的唯一证书。
你可以使用 `kubeadm kubeconfig user --client-name <CN>`

命令执行此操作。
该命令会将 KubeConfig 文件打印到 STDOUT，你应该将其保存到文件并分发给用户。
之后，使用 `kubectl create (cluster)rolebinding`

授予特权。

如果你要从集群外部连接到 API 服务器，则可以使用 `kubectl proxy`

：

```
scp root@<control-plane-host>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf proxy
```


你现在可以在 `http://localhost:8001/api/v1`

从本地访问 API 服务器。

如果你在集群中使用了一次性服务器进行测试，则可以关闭这些服务器，而无需进一步清理。
你可以使用 `kubectl config delete-cluster`

删除对集群的本地引用。

但是，如果要更干净地取消配置集群， 则应首先清空节点并确保该节点为空， 然后取消配置该节点。

使用适当的凭据与控制平面节点通信，运行：

```
kubectl drain <节点名称> --delete-emptydir-data --force --ignore-daemonsets
```


在移除节点之前，请重置 `kubeadm`

安装的状态：

```
kubeadm reset
```


重置过程不会重置或清除 iptables 规则或 IPVS 表。如果你希望重置 iptables，则必须手动进行：

```
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```


如果要重置 IPVS 表，则必须运行以下命令：

```
ipvsadm -C
```


现在移除节点：

```
kubectl delete node <节点名称>
```


如果你想重新开始，只需运行 `kubeadm init`

或 `kubeadm join`

并加上适当的参数。

你可以在控制平面主机上使用 `kubeadm reset`

来触发尽力而为的清理。

有关此子命令及其选项的更多信息，请参见
`kubeadm reset`

参考文档。

虽然 kubeadm 允许所管理的组件有一定程度的版本偏差， 但是建议你将 kubeadm 的版本与控制平面组件、kube-proxy 和 kubelet 的版本相匹配。

kubeadm 可以与 Kubernetes 组件一起使用，这些组件的版本与 kubeadm 相同，或者比它大一个版本。
Kubernetes 版本可以通过使用 `--kubeadm init`

的 `--kubernetes-version`

标志或使用 `--config`

时的
`ClusterConfiguration.kubernetesVersion`

字段指定给 kubeadm。
这个选项将控制 kube-apiserver、kube-controller-manager、kube-scheduler 和 kube-proxy 的版本。

例子：

- kubeadm 的版本为 1.36。
`kubernetesVersion`

必须为 1.36 或者 1.35。

与 Kubernetes 版本类似，kubeadm 可以使用与 kubeadm 相同版本的 kubelet， 或者比 kubeadm 老三个版本的 kubelet。

例子：

- kubeadm 的版本为 1.36。
- 主机上的 kubelet 必须为 1.36、1.35、 1.34 或 1.33。

kubeadm 命令在现有节点或由 kubeadm 管理的整个集群上的操作有一定限制。

如果新的节点加入到集群中，用于 `kubeadm join`

的 kubeadm 二进制文件必须与用 `kubeadm init`

创建集群或用 `kubeadm upgrade`

升级同一节点时所用的 kubeadm 版本一致。
类似的规则适用于除了 `kubeadm upgrade`

以外的其他 kubeadm 命令。

`kubeadm join`

的例子：

- 使用
`kubeadm init`

创建集群时使用版本为 1.36 的 kubeadm。 - 添加节点所用的 kubeadm 可执行文件为版本 。

对于正在升级的节点，所使用的的 kubeadm 必须与管理该节点的 kubeadm 具有相同的 MINOR 版本或比后者新一个 MINOR 版本。

`kubeadm upgrade`

的例子：

- 用于创建或升级节点的 kubeadm 版本为 1.35。
- 用于升级节点的 kubeadm 版本必须为 1.35 或 1.36。

要了解更多关于不同 Kubernetes 组件之间的版本偏差， 请参见版本偏差策略。

此处创建的集群具有单个控制平面节点，运行单个 etcd 数据库。 这意味着如果控制平面节点发生故障，你的集群可能会丢失数据并且可能需要从头开始重新创建。

解决方法：

- 定期备份 etcd。
kubeadm 配置的 etcd 数据目录位于控制平面节点上的
`/var/lib/etcd`

中。

kubeadm deb/rpm 软件包和二进制文件是为 amd64、arm (32-bit)、arm64、ppc64le 和 s390x 构建的遵循多平台提案。

从 v1.12 开始还支持用于控制平面和附加组件的多平台容器镜像。

只有一些网络提供商为所有平台提供解决方案。 请查阅上方的网络提供商清单或每个提供商的文档以确定提供商是否支持你选择的平台。

如果你在使用 kubeadm 时遇到困难， 请查阅我们的故障排除文档。

- 使用 Sonobuoy 验证集群是否正常运行。
- 有关使用 kubeadm 升级集群的详细信息， 请参阅升级 kubeadm 集群。
- 在 kubeadm 参考文档中了解有关
`kubeadm`

进阶用法的信息。 - 了解有关 Kubernetes 概念和
`kubectl`

的更多信息。 - 有关 Pod 网络附加组件的更多列表，请参见集群网络页面。
- 请参阅附加组件列表以探索其他附加组件， 包括用于 Kubernetes 集群的日志记录、监视、网络策略、可视化和控制的工具。
- 配置集群如何处理集群事件的日志以及在 Pod 中运行的应用程序。 有关所涉及内容的概述，请参见日志架构。

- 有关漏洞，访问 kubeadm GitHub issue tracker
- 有关支持，访问 #kubeadm Slack 频道
- 常规的 SIG Cluster Lifecycle 开发 Slack 频道： #sig-cluster-lifecycle
- SIG Cluster Lifecycle 的 SIG 资料
- SIG Cluster Lifecycle 邮件列表： kubernetes-sig-cluster-lifecycle

最后修改 April 21, 2026 at 9:16 AM PST: [zh-cn]sync node-conformance create-cluster-kubeadm (9099459cf6)