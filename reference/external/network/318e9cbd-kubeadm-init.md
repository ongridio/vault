---
title: kubeadm init
source: https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/
kind: external
domain: network
original_date: 2025-12-19
fetched_at: 2026-05-16
bookmark_title: kubeadm init | Kubernetes
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[kubernetes.io](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)
> 原始日期：2025-12-19
> 抓取日期：2026-05-16

# kubeadm init

此命令初始化一个 Kubernetes 控制平面节点。

运行此命令来搭建 Kubernetes 控制平面节点。

"init" 命令执行以下阶段：

```
preflight 预检
certs 生成证书
/ca 生成自签名根 CA 用于配置其他 kubernetes 组件
/apiserver 生成 apiserver 的证书
/apiserver-kubelet-client 生成 apiserver 连接到 kubelet 的证书
/front-proxy-ca 生成前端代理自签名 CA（扩展apiserver）
/front-proxy-client 生成前端代理客户端的证书（扩展 apiserver）
/etcd-ca 生成 etcd 自签名 CA
/etcd-server 生成 etcd 服务器证书
/etcd-peer 生成 etcd 节点相互通信的证书
/etcd-healthcheck-client 生成 etcd 健康检查的证书
/apiserver-etcd-client 生成 apiserver 访问 etcd 的证书
/sa 生成用于签署服务帐户令牌的私钥和公钥
kubeconfig 生成建立控制平面和管理所需的所有 kubeconfig 文件
/admin 生成一个 kubeconfig 文件供管理员使用以及供 kubeadm 本身使用
/super-admin 为超级管理员生成 kubeconfig 文件
/kubelet 为 kubelet 生成一个 kubeconfig 文件，*仅*用于集群引导
/controller-manager 生成 kubeconfig 文件供控制器管理器使用
/scheduler 生成 kubeconfig 文件供调度程序使用
etcd 为本地 etcd 生成静态 Pod 清单文件
/local 为本地单节点本地 etcd 实例生成静态 Pod 清单文件
control-plane 生成建立控制平面所需的所有静态 Pod 清单文件
/apiserver 生成 kube-apiserver 静态 Pod 清单
/controller-manager 生成 kube-controller-manager 静态 Pod 清单
/scheduler 生成 kube-scheduler 静态 Pod 清单
kubelet-start 写入 kubelet 设置并启动（或重启）kubelet
wait-control-plane 等待控制平面启动
upload-config 将 kubeadm 和 kubelet 配置上传到 ConfigMap
/kubeadm 将 kubeadm 集群配置上传到 ConfigMap
/kubelet 将 kubelet 组件配置上传到 ConfigMap
upload-certs 将证书上传到 kubeadm-certs
mark-control-plane 将节点标记为控制面
bootstrap-token 生成用于将节点加入集群的引导令牌
kubelet-finalize 在 TLS 引导后更新与 kubelet 相关的设置
addon 安装用于通过一致性测试所需的插件
/coredns 将 CoreDNS 插件安装到 Kubernetes 集群
/kube-proxy 将 kube-proxy 插件安装到 Kubernetes 集群
show-join-command 显示控制平面和工作节点的加入命令
```


```
kubeadm init [flags]
```


| --apiserver-advertise-address string | |
API 服务器所公布的其正在监听的 IP 地址。如果未设置，则使用默认网络接口。 | |
| --apiserver-bind-port int32 默认值：6443 | |
API 服务器绑定的端口。 | |
| --apiserver-cert-extra-sans strings | |
用于 API Server 服务证书的可选附加主题备用名称（SAN）。 可以是 IP 地址和 DNS 名称。 | |
| --cert-dir string 默认值："/etc/kubernetes/pki" | |
保存和存储证书的路径。 | |
| --certificate-key string | |
用于加密 kubeadm-certs Secret 中的控制平面证书的密钥。 证书密钥为十六进制编码的字符串，是大小为 32 字节的 AES 密钥。 | |
| --config string | |
kubeadm 配置文件的路径。 | |
| --control-plane-endpoint string | |
为控制平面指定一个稳定的 IP 地址或 DNS 名称。 | |
| --cri-socket string | |
要连接的 CRI 套接字的路径。如果为空，则 kubeadm 将尝试自动检测此值； 仅当安装了多个 CRI 或具有非标准 CRI 套接字时，才使用此选项。 | |
| --dry-run | |
不做任何更改；只输出将要执行的操作。 | |
| --feature-gates string | |
一组用来描述各种特性门控的键值（key=value）对。选项是： | |
| -h, --help | |
init 操作的帮助命令。 | |
| --ignore-preflight-errors strings | |
错误将显示为警告的检查列表；例如：'IsPrivilegedUser,Swap'。 取值为 'all' 时将忽略检查中的所有错误。 | |
| --image-repository string 默认值："registry.k8s.io" | |
选择用于拉取控制平面镜像的容器仓库。 | |
| --kubernetes-version string 默认值："stable-1" | |
为控制平面选择一个特定的 Kubernetes 版本。 | |
| --node-name string | |
指定节点的名称。 | |
| --patches string | |
它包含名为 "target[suffix][+patchtype].extension" 的文件的目录的路径。 例如，"kube-apiserver0+merge.yaml" 或仅仅是 "etcd.json"。 "target" 可以是 "kube-apiserver"、"kube-controller-manager"、"kube-scheduler"、 "etcd"、"kubeletconfiguration" 之一。 "patchtype" 可以是 "strategic"、"merge" 或者 "json" 之一， 并且它们与 kubectl 支持的补丁格式相同。 默认的 "patchtype" 是 "strategic"。 "extension" 必须是 "json" 或 "yaml"。 "suffix" 是一个可选字符串，可用于确定首先按字母顺序应用哪些补丁。 | |
| --pod-network-cidr string | |
指明 Pod 网络可以使用的 IP 地址段。如果设置了这个参数， 控制平面将会为每一个节点自动分配 CIDR。 | |
| --service-cidr string 默认值："10.96.0.0/12" | |
为服务的虚拟 IP 地址另外指定 IP 地址段。 | |
| --service-dns-domain string 默认值："cluster.local" | |
为服务另外指定域名，例如："myorg.internal"。 | |
| --skip-certificate-key-print | |
不要打印用于加密控制平面证书的密钥。 | |
| --skip-phases strings | |
要跳过的阶段列表。 | |
| --skip-token-print | |
跳过打印 'kubeadm init' 生成的默认引导令牌。 | |
| --token string | |
这个令牌用于建立控制平面节点与工作节点间的双向通信。
格式为 | |
| --token-ttl duration 默认值：24h0m0s | |
令牌被自动删除之前的持续时间（例如 1s，2m，3h）。如果设置为 '0'，则令牌将永不过期。 | |
| --upload-certs | |
将控制平面证书上传到 kubeadm-certs Secret。 |

| --rootfs string | |
[实验] 到'真实'主机根文件系统的路径。 |

`kubeadm init`

命令通过执行下列步骤来启动一个 Kubernetes 控制平面节点。

- 在做出变更前运行一系列的预检项来验证系统状态。一些检查项目仅仅触发警告，
其它的则会被视为错误并且退出 kubeadm，除非问题得到解决或者用户指定了
`--ignore-preflight-errors=<错误列表>`

参数。

- 生成一个自签名的 CA 证书来为集群中的每一个组件建立身份标识。
用户可以通过将其放入
`--cert-dir`

配置的证书目录中（默认为`/etc/kubernetes/pki`

） 来提供他们自己的 CA 证书以及/或者密钥。 API 服务器证书将为所有`--apiserver-cert-extra-sans`

参数值提供附加的 SAN 条目，必要时将其小写。

- 将 kubeconfig 文件写入
`/etc/kubernetes/`

目录以便 kubelet、控制器管理器和调度器连接到 API 服务器，它们每一个都有自己的身份标识。再编写额外的 kubeconfig 文件，将 kubeadm 作为管理实体（`admin.conf`

）和可以绕过 RBAC 的超级管理员用户（`super-admin.conf`

）。

为 API 服务器、控制器管理器和调度器生成静态 Pod 的清单文件。假使没有提供一个外部的 etcd 服务的话，也会为 etcd 生成一份额外的静态 Pod 清单文件。

静态 Pod 的清单文件被写入到

`/etc/kubernetes/manifests`

目录； kubelet 会监视这个目录以便在系统启动的时候创建 Pod。一旦控制平面的 Pod 都运行起来，

`kubeadm init`

的工作流程就继续往下执行。

- 对控制平面节点应用标签和污点标记以便不会在它上面运行其它的工作负载。

- 生成令牌，将来其他节点可使用该令牌向控制平面注册自己。如
kubeadm token
文档所述，用户可以选择通过
`--token`

提供令牌。

为了使得节点能够遵照启动引导令牌和 TLS 启动引导 这两份文档中描述的机制加入到集群中，kubeadm 会执行所有的必要配置：

创建一个 ConfigMap 提供添加集群节点所需的信息，并为该 ConfigMap 设置相关的 RBAC 访问规则。

允许启动引导令牌访问 CSR 签名 API。

配置自动签发新的 CSR 请求。


更多相关信息，请查看 kubeadm join。


通过 API 服务器安装一个 DNS 服务器（CoreDNS）和 kube-proxy 附加组件。 在 Kubernetes v1.11 和更高版本中，CoreDNS 是默认的 DNS 服务器。 请注意，尽管已部署 DNS 服务器，但直到安装 CNI 时才调度它。

#### 警告：

从 v1.18 开始，在 kubeadm 中使用 kube-dns 的支持已被废弃，并已在 v1.21 版本中移除。


kubeadm 允许你使用 `kubeadm init phase`

命令分阶段创建控制平面节点。

要查看阶段和子阶段的有序列表，可以调用 `kubeadm init --help`

。
该列表将位于帮助屏幕的顶部，每个阶段旁边都有一个描述。
注意，通过调用 `kubeadm init`

，所有阶段和子阶段都将按照此确切顺序执行。

某些阶段具有唯一的标志，因此，如果要查看可用选项的列表，请添加 `--help`

，例如：

```
sudo kubeadm init phase control-plane controller-manager --help
```


你也可以使用 `--help`

查看特定父阶段的子阶段列表：

```
sudo kubeadm init phase control-plane --help
```


`kubeadm init`

还公开了一个名为 `--skip-phases`

的参数，该参数可用于跳过某些阶段。
参数接受阶段名称列表，并且这些名称可以从上面的有序列表中获取。

例如：

```
sudo kubeadm init phase control-plane all --config=configfile.yaml
sudo kubeadm init phase etcd local --config=configfile.yaml
# 你现在可以修改控制平面和 etcd 清单文件
sudo kubeadm init --skip-phases=control-plane,etcd --config=configfile.yaml
```


该示例将执行的操作是基于 `configfile.yaml`

中的配置在 `/etc/kubernetes/manifests`

中写入控制平面和 etcd 的清单文件。
这允许你修改文件，然后使用 `--skip-phases`

跳过这些阶段。
通过调用最后一个命令，你将使用自定义清单文件创建一个控制平面节点。

特性状态：

`Kubernetes v1.22 [beta]`

或者，你可以使用 `InitConfiguration`

下的 `skipPhases`

字段。

配置文件的功能仍然处于 Beta 状态并且在将来的版本中可能会改变。

通过一份配置文件而不是使用命令行参数来配置 `kubeadm init`

命令是可能的，
但是一些更加高级的功能只能够通过配置文件设定。
这份配置文件通过 `--config`

选项参数指定的，
它必须包含 `ClusterConfiguration`

结构，并可能包含更多由 `---\n`

分隔的结构。
在某些情况下，可能不允许将 `--config`

与其他标志混合使用。

可以使用 kubeadm config print 命令打印出默认配置。

如果你的配置没有使用最新版本，**推荐**使用
kubeadm config migrate
命令进行迁移。

关于配置的字段和用法的更多信息，你可以访问 API 参考页面。

kubeadm 支持一组独有的特性门控，只能在 `kubeadm init`

创建集群期间使用。
这些特性可以控制集群的行为。特性门控会在毕业到 GA 后被移除。

你可以使用 `--feature-gates`

标志来为 `kubeadm init`

设置特性门控，
或者你可以在用 `--config`

传递配置文件时添加条目到
`featureGates`

字段中。

直接传递 Kubernetes 核心组件的特性门控给 kubeadm 是不支持的。 相反，可以通过使用 kubeadm API 的自定义组件来传递。

特性门控的列表：

| 特性 | 默认值 | Alpha | Beta | GA |
|---|---|---|---|---|
`ControlPlaneKubeletLocalMode` | `true` | 1.31 | 1.33 | - |
`NodeLocalCRISocket` | `true` | 1.32 | 1.34 | - |

一旦特性门控变成了 GA，它的值会被默认锁定为 `true`

。

特性门控的描述：

`ControlPlaneKubeletLocalMode`

- 启用此特性门控后，当加入新的控制平面节点时， kubeadm 将配置 kubelet 连接到本地 kube-apiserver。 这将确保在滚动升级期间不会违反版本偏差策略。

`NodeLocalCRISocket`

- 启用此特性门控后，kubeadm 将使用
`/var/lib/kubelet/instance-config.yaml`

文件读写每个节点的 CRI 套接字， 不再是从 Node 对象上的`kubeadm.alpha.kubernetes.io/cri-socket`

注解读取 CRI 套接字， 也不再将 CRI 套接字写入到 Node 对象的`kubeadm.alpha.kubernetes.io/cri-socket`

注解。 这个新的文件将作为实例配置补丁被应用，之后才会应用其他通过`--patches`

标志设置的用户管理的补丁。 这个新的文件仅包含源自 KubeletConfiguration 文件格式的字段`containerRuntimeEndpoint`

。如果升级期间此特性门控被启用，但`/var/lib/kubelet/instance-config.yaml`

文件还不存在，kubeadm 将尝试从`/var/lib/kubelet/kubeadm-flags.env`

文件读取 CRI 套接字值。

已弃用特性门控的列表：

| 特性 | 默认值 | Alpha | Beta | GA | 弃用 |
|---|---|---|---|---|---|
`PublicKeysECDSA` | `false` | 1.19 | - | - | 1.31 |
`RootlessControlPlane` | `false` | 1.22 | - | - | 1.31 |

特性门控描述：

`PublicKeysECDSA`

- 可用于创建一个使用 ECDSA 证书而非默认 RSA 算法的集群。
支持用
`kubeadm certs renew`

更新现有 ECDSA 证书， 但你不能在集群运行期间或升级期间切换 RSA 和 ECDSA 算法。 在 v1.31 之前的 Kubernetes 版本中有一个 Bug，即使你启用了`PublicKeysECDSA`

特性门控， 所生成的 kubeconfig 文件中的密钥仍然使用 RSA 设置。 此特性门控现已弃用，替换为 kubeadm v1beta4 中可用的`encryptionAlgorithm`

功能。

`RootlessControlPlane`

- 设置此标志来配置 kubeadm 所部署的控制平面组件中的静态 Pod 容器
`kube-apiserver`

、`kube-controller-manager`

、`kube-scheduler`

和`etcd`

以非 root 用户身份运行。如果未设置该标志，则这些组件以 root 身份运行。 你可以在升级到更新版本的 Kubernetes 之前更改此特性门控的值。

已移除的特性门控列表：

| 特性 | Alpha | Beta | GA | 移除 |
|---|---|---|---|---|
`EtcdLearnerMode` | 1.27 | 1.29 | 1.32 | 1.33 |
`IPv6DualStack` | 1.16 | 1.21 | 1.23 | 1.24 |
`UnversionedKubeletConfigMap` | 1.22 | 1.23 | 1.25 | 1.26 |
`UpgradeAddonsBeforeControlPlane` | 1.28 | - | - | 1.31 |
`WaitForAllControlPlaneComponents` | 1.30 | 1.33 | 1.34 | 1.35 |

特性门控的描述：

`EtcdLearnerMode`

- 当加入一个新的控制平面节点时，会创建一个新的 etcd 成员作为 learner， 并且仅在 etcd 数据完全对齐后，才会将其提升为投票成员。

`IPv6DualStack`

- 在 IP 双栈特性处于开发过程中时，此标志有助于配置组件的双栈支持。有关 Kubernetes 双栈支持的更多详细信息，请参阅 kubeadm 的双栈支持。

`UnversionedKubeletConfigMap`

- 此标志控制 kubeadm 存储 kubelet 配置数据的 ConfigMap 的名称。
在未指定此标志或设置为
`true`

的情况下，此 ConfigMap 被命名为`kubelet-config`

。 如果将此标志设置为`false`

，则此 ConfigMap 的名称会包括 Kubernetes 的主要版本和次要版本 （例如：`kubelet-config-1.36`

）。 kubeadm 会确保用于读写 ConfigMap 的 RBAC 规则适合你设置的值。 当 kubeadm 写入此 ConfigMap 时（在`kubeadm init`

或`kubeadm upgrade apply`

期间）， kubeadm 根据`UnversionedKubeletConfigMap`

的设置值来执行操作。 当读取此 ConfigMap 时（在执行`kubeadm join`

、`kubeadm reset`

、`kubeadm upgrade`

等操作期间）， kubeadm 尝试首先使用无版本（后缀）的 ConfigMap 名称； 如果不成功，kubeadm 将回退到使用该 ConfigMap 的旧（带版本号的）名称。

`UpgradeAddonsBeforeControlPlane`

- 此特性门控已被移除。它在 v1.28 中作为一个已弃用的特性被引入，在 v1.31 中被移除。 有关旧版本的文档，请切换到相应的网站版本。

`WaitForAllControlPlaneComponents`

- 启用此特性门控后，kubeadm 将等待控制平面节点上的所有控制平面组件
（kube-apiserver、kube-controller-manager、kube-scheduler）在其
`/livez`

或`/healthz`

端点上报告 200 状态码。这些检测请求是针对`https://ADDRESS:PORT/ENDPOINT`

进行的。其中：`PORT`

取自组件的`--secure-port`

标志。`ADDRESS`

对 kube-apiserver 而言是其`--advertise-address`

，对于 kube-scheduler 和 kube-controller-manager 而言是其`--bind-address`

。- 对于 kube-controller-manager，其
`ENDPOINT`

只能是`/healthz`

，直到它也支持`/livez`

为止。

如果你在 kubeadm 配置中指定自定义的

`ADDRESS`

或`PORT`

，kubeadm 将使用这些定制的值。 如果没有启用此特性门控，kubeadm 将仅等待控制平面节点上的 kube-apiserver 准备就绪。 等待过程在 kubeadm 启动主机上的 kubelet 后立即开始。如果你希望在`kubeadm init`

或`kubeadm join`

命令执行期间观察所有控制平面组件的就绪状态，建议你启用此特性门控。

kubeadm 配置中有关 kube-proxy 的说明请查看：

使用 kubeadm 启用 IPVS 模式的说明请查看：

有关向控制平面组件传递命令行参数的说明请查看：

要在没有互联网连接的情况下运行 kubeadm，你必须提前拉取所需的控制平面镜像。

你可以使用 `kubeadm config images`

子命令列出并拉取镜像：

```
kubeadm config images list
kubeadm config images pull
```


你可以通过 `--config`

把 kubeadm 配置文件传递给上述命令来控制
`kubernetesVersion`

和 `imageRepository`

字段。

kubeadm 需要的所有默认 `registry.k8s.io`

镜像都支持多种硬件体系结构。

默认情况下，kubeadm 会从 `registry.k8s.io`

仓库拉取镜像。如果请求的 Kubernetes 版本是 CI
标签（例如 `ci/latest`

），则使用 `gcr.io/k8s-staging-ci-images`

。

你可以通过使用带有配置文件的 kubeadm 来重写此操作。 允许的自定义功能有：

- 提供影响镜像版本的
`kubernetesVersion`

。 - 使用其他的
`imageRepository`

来代替`registry.k8s.io`

。 - 为 etcd 或 CoreDNS 提供特定的
`imageRepository`

和`imageTag`

。

由于向后兼容的原因，使用 `imageRepository`

所指定的定制镜像库可能与默认的
`registry.k8s.io`

镜像路径不同。例如，某镜像的子路径可能是 `registry.k8s.io/subpath/image`

，
但使用自定义仓库时默认为 `my.customrepository.io/image`

。

确保将镜像推送到 kubeadm 可以使用的自定义仓库的路径中，你必须：

- 使用
`kubeadm config images {list|pull}`

从`registry.k8s.io`

的默认路径中拉取镜像。 - 将镜像推送到
`kubeadm config images list --config=config.yaml`

的路径， 其中`config.yaml`

包含自定义的`imageRepository`

和/或用于 etcd 和 CoreDNS 的`imageTag`

。 - 将相同的
`config.yaml`

传递给`kubeadm init`

。

如果需要为这些组件设置定制的镜像， 你需要在你的容器运行时中完成一些配置。 参阅你的容器运行时的文档以了解如何改变此设置。 对于某些容器运行时而言， 你可以在容器运行时主题下找到一些建议。

通过将参数 `--upload-certs`

添加到 `kubeadm init`

，你可以将控制平面证书临时上传到集群中的 Secret。
请注意，此 Secret 将在 2 小时后自动过期。这些证书使用 32 字节密钥加密，可以使用 `--certificate-key`

指定该密钥。
通过将 `--control-plane`

和 `--certificate-key`

传递给 `kubeadm join`

，
可以在添加其他控制平面节点时使用相同的密钥下载证书。

以下阶段命令可用于证书到期后重新上传证书：

```
kubeadm init phase upload-certs --upload-certs --config=SOME_YAML_FILE
```


在使用 `--config`

传递配置文件时，
可以在 `InitConfiguration`

中提供预定义的 `certificateKey`

。

如果未将预定义的证书密钥传递给 `kubeadm init`

和 `kubeadm init phase upload-certs`

，
则会自动生成一个新密钥。

以下命令可用于按需生成新密钥：

```
kubeadm certs certificate-key
```


有关使用 kubeadm 进行证书管理的详细信息， 请参阅使用 kubeadm 进行证书管理。 该文档包括有关使用外部 CA、自定义证书和证书续订的信息。

`kubeadm`

包自带了关于 `systemd`

如何运行 `kubelet`

的配置文件。
请注意 `kubeadm`

客户端命令行工具永远不会修改这份 `systemd`

配置文件。
这份 `systemd`

配置文件属于 kubeadm DEB/RPM 包。

有关更多信息， 请阅读管理 systemd 的 kubeadm 内嵌文件。

默认情况下，kubeadm 尝试检测你的容器运行环境。有关此检测的更多详细信息，请参见 kubeadm CRI 安装指南。

默认情况下，kubeadm 基于机器的主机地址分配一个节点名称。你可以使用 `--node-name`

参数覆盖此设置。
此标识将合适的 `--hostname-override`

值传递给 kubelet。

要注意，重载主机名可能会与云驱动发生冲突。

除了像文档
kubeadm 基础教程中所描述的那样，
将从 `kubeadm init`

取得的令牌复制到每个节点，你还可以并行地分发令牌以实现更简单的自动化。
要实现自动化，你必须知道控制平面节点启动后将拥有的 IP 地址，或使用 DNS 名称或负载均衡器的地址。

生成一个令牌。这个令牌必须采用的格式为：

`<6 个字符的字符串>.<16 个字符的字符串>`

。 更加正式的说法是，它必须符合正则表达式：`[a-z0-9]{6}\.[a-z0-9]{16}`

。kubeadm 可以为你生成一个令牌：

`kubeadm token generate`


- 使用这个令牌同时启动控制平面节点和工作节点。这些节点一旦运行起来应该就会互相寻找对方并且形成集群。
同样的
`--token`

参数可以同时用于`kubeadm init`

和`kubeadm join`

命令。

当接入其他控制平面节点时，可以对

`--certificate-key`

执行类似的操作。可以使用以下方式生成密钥：`kubeadm certs certificate-key`


一旦集群启动起来，你就可以从控制平面节点的 `/etc/kubernetes/admin.conf`

文件获取管理凭据，
并使用这个凭据同集群通信。

一旦集群启动起来，你就可以从控制平面节点中的 `/etc/kubernetes/admin.conf`

文件获取管理凭证或通过为其他用户生成的 kubeconfig 文件与集群通信。

注意这种搭建集群的方式在安全保证上会有一些宽松，因为这种方式不允许使用
`--discovery-token-ca-cert-hash`

来验证根 CA
的哈希值（因为当配置节点的时候，它还没有被生成）。
更多信息请参阅 kubeadm join 文档。

- 进一步阅读了解 kubeadm init 阶段
- kubeadm join 启动一个 Kubernetes 工作节点并且将其加入到集群
- kubeadm upgrade 将 Kubernetes 集群升级到新版本
- kubeadm reset
恢复
`kubeadm init`

或`kubeadm join`

命令对节点所作的变更

最后修改 December 19, 2025 at 4:22 PM PST: [zh-cn]sync virtual-ips kubeadm-init (c99a0a2db9)