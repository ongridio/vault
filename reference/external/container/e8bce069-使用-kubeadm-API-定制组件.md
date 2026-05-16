---
title: 使用 kubeadm API 定制组件
source: https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/control-plane-flags/
kind: external
domain: container
original_date: 2026-05-06
fetched_at: 2026-05-16
bookmark_title: 使用 kubeadm 定制控制平面配置 | Kubernetes
tags: [external, container]
---

> [!info] 外部文章 · 自动导入
> 来源：[kubernetes.io](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/control-plane-flags/)
> 原始日期：2026-05-06
> 抓取日期：2026-05-16

# 使用 kubeadm API 定制组件

本页面介绍了如何自定义 kubeadm 部署的组件。
你可以使用 `ClusterConfiguration`

结构中定义的参数，或者在每个节点上应用补丁来定制控制平面组件。
你可以使用 `KubeletConfiguration`

和 `KubeProxyConfiguration`

结构分别定制 kubelet 和 kube-proxy 组件。

所有这些选项都可以通过 kubeadm 配置 API 实现。 有关配置中的每个字段的详细信息，你可以导航到我们的 API 参考页面 。

要重新配置已创建的集群，请参阅重新配置 kubeadm 集群。

`ClusterConfiguration`

中的标志自定义控制平面kubeadm `ClusterConfiguration`

对象为用户提供了一种方法，
用以覆盖传递给控制平面组件（如 APIServer、ControllerManager、Scheduler 和 Etcd）的默认参数。
各组件配置使用如下字段定义：

`apiServer`

`controllerManager`

`scheduler`

`etcd`


这些结构包含一个通用的 `extraArgs`

字段，该字段由 `name`

/ `value`

组成。
要覆盖控制平面组件的参数：

- 将适当的字段
`extraArgs`

添加到配置中。 - 向字段
`extraArgs`

添加要覆盖的参数值。 - 用
`--config <YOUR CONFIG YAML>`

运行`kubeadm init`

。

你可以通过运行 `kubeadm config print init-defaults`

并将输出保存到你所选的文件中，
以默认值形式生成 `ClusterConfiguration`

对象。

`ClusterConfiguration`

对象目前在 kubeadm 集群中是全局的。
这意味着你添加的任何标志都将应用于同一组件在不同节点上的所有实例。
要在不同节点上为每个组件应用单独的配置，你可以使用补丁。

当前不支持重复的参数（keys）或多次传递相同的参数 `--foo`

。
要解决此问题，你必须使用补丁。

有关详细信息，请参阅 kube-apiserver 参考文档。

使用示例：

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
apiServer:
extraArgs:
- name: "enable-admission-plugins"
value: "AlwaysPullImages,DefaultStorageClass"
- name: "audit-log-path"
value: "/home/johndoe/audit.log"
```


有关详细信息，请参阅 kube-controller-manager 参考文档。

使用示例：

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
controllerManager:
extraArgs:
- name: "cluster-signing-key-file"
value: "/home/johndoe/keys/ca.key"
- name: "deployment-controller-sync-period"
value: "50"
```


有关详细信息，请参阅 kube-scheduler 参考文档。

使用示例：

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
scheduler:
extraArgs:
- name: "config"
value: "/etc/kubernetes/scheduler-config.yaml"
extraVolumes:
- name: schedulerconfig
hostPath: /home/johndoe/schedconfig.yaml
mountPath: /etc/kubernetes/scheduler-config.yaml
readOnly: true
pathType: "File"
```


有关详细信息，请参阅 Etcd 服务文档。

使用示例：

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
etcd:
local:
extraArgs:
- name: "election-timeout"
value: 1000
```


特性状态：

`Kubernetes v1.22 [beta]`

Kubeadm 允许将包含补丁文件的目录传递给各个节点上的
`InitConfiguration`

、`JoinConfiguration`

和 `UpgradeConfiguration`

。
这些补丁可被用作组件配置写入磁盘之前的最后一个自定义步骤。

可以使用 `--config <你的 YAML 格式控制文件>`

将配置文件传递给 `kubeadm init`

：

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
patches:
directory: /home/user/somedir
```


如果你使用 `kubeadm upgrade apply`

和 `kubeadm upgrade node`

来升级你的 kubeadm 节点，你必须再次提供相同的补丁，以便在升级后保留自定义设置。

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: UpgradeConfiguration
apply:
patches:
directory: /home/user/somedir
```


```
apiVersion: kubeadm.k8s.io/v1beta4
kind: UpgradeConfiguration
node:
patches:
directory: /home/user/somedir
```


对于 `kubeadm init`

，你可以传递一个包含 `ClusterConfiguration`

和
`InitConfiguration`

的文件，以 `---`

分隔。

你可以使用 `--config <你的 YAML 格式配置文件>`

将配置文件传递给 `kubeadm join`

：

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
patches:
directory: /home/user/somedir
```


补丁目录必须包含名为 `target[suffix][+patchtype].extension`

的文件。
例如，`kube-apiserver0+merge.yaml`

或只是 `etcd.json`

。

`target`

可以是`kube-apiserver`

、`kube-controller-manager`

、`kube-scheduler`

、`etcd`

、`kubeletconfiguration`

和`kubeletconfiguration`

之一。`suffix`

是一个可选字符串，可用于确定首先按字母数字应用哪些补丁。`patchtype`

可以是`strategy`

、`merge`

或`json`

之一，并且这些必须匹配 kubectl 支持 的补丁格式。默认补丁类型是`strategic`

的。`extension`

必须是`json`

或`yaml`

。

要自定义 kubelet，你可以在同一配置文件中的 `ClusterConfiguration`

或 `InitConfiguration`

之外添加一个 `KubeletConfiguration`

，
用 `---`

分隔。然后可以将此文件传递给 `kubeadm init`

，kubeadm 会将相同的
`KubeletConfiguration`

配置应用于集群中的所有节点。

要在基础 `KubeletConfiguration`

上应用特定节点的配置，你可以使用
`kubeletconfiguration`

补丁定制。

或者你可以使用 `kubelet`

参数进行覆盖，方法是将它们传递到 `InitConfiguration`

和 `JoinConfiguration`

支持的 `nodeRegistration.kubeletExtraArgs`

字段中。一些 kubelet 参数已被弃用，
因此在使用这些参数之前，请在
kubelet 参考文档中检查它们的状态。

更多详情，请参阅使用 kubeadm 配置集群中的每个 kubelet。

要自定义 kube-proxy，你可以在 `ClusterConfiguration`

或 `InitConfiguration`

之外添加一个由 `---`

分隔的 `KubeProxyConfiguration`

，传递给 `kubeadm init`

。

可以导航到 API 参考页面查看更多详情。

kubeadm 将 kube-proxy 部署为 DaemonSet，
这意味着 `KubeProxyConfiguration`

将应用于集群中的所有 kube-proxy 实例。

kubeadm 允许你通过针对
`corednsdeployment`

补丁目标的补丁来定制 CoreDNS Deployment。

目前不支持对其他 CoreDNS 相关 API 对象（如 `kube-system/coredns`

ConfigMap）的补丁。
你需要使用 kubectl 手动修补这些对象，
并在之后重新创建 CoreDNS Pod。

或者，你可以通过在 `ClusterConfiguration`

中包含以下选项来禁用 kubeadm CoreDNS 部署：

```
dns:
disabled: true
```


另外，通过执行以下命令：

```
kubeadm init phase addon coredns --print-manifest --config my-config.yaml`
```


你可以获取 kubeadm 在你的设置中为 CoreDNS 创建的清单文件。

最后修改 May 06, 2026 at 6:30 AM PST: change H2 headings to H3 to match English source - part 2 (ea1f474de5)