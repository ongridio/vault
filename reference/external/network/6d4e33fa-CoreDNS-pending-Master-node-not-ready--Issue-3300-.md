---
title: CoreDNS pending; Master node not ready · Issue #3300 · coredns/coredns
source: https://github.com/coredns/coredns/issues/3300
kind: external
domain: network
author: Coredns
original_date: 2019-09-24
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[github.com](https://github.com/coredns/coredns/issues/3300)
> 作者：Coredns
> 原始日期：2019-09-24
> 抓取日期：2026-05-16

# CoreDNS pending; Master node not ready · Issue #3300 · coredns/coredns

-
Notifications
You must be signed in to change notification settings - Fork 2.4k

# CoreDNS pending; Master node not ready #3300

Copy link

Copy link

Closed

Labels

## Description

Hello I have Centos-7.7x86_64 with 2 ethernet interfaces.

eth0 - 172.xxx.xxx.xxx

eth1- 10.2.10.7

I installed kubernetes but My master node have status not ready, and coredns have status Pending.

I did in first time

kubeadm init --pod-network-cidr=172.17.0.1/16 and I had that problem

in second time i did

kubeadm init --apiserver-advertise-address=10.2.10.7 --pod-network-cidr=172.17.0.1/16

and I have that problem

And YEs I did

mkdir -p $HOME/.kube

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

chown

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

Out of command

```
[root@pr02 ~]# kubectl get pods --all-namespaces -o wide
NAMESPACE NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
kube-system coredns-5644d7b6d9-fl7h8 0/1 Pending 0 25m <none> <none> <none> <none>
kube-system coredns-5644d7b6d9-rvh6q 0/1 Pending 0 25m <none> <none> <none> <none>
kube-system etcd-pr02 1/1 Running 0 24m 178.xxx.xxx.xxx pr02 <none> <none>
kube-system kube-apiserver-pr02 1/1 Running 0 24m 178.xxx.xxx.xxx pr02 <none> <none>
kube-system kube-controller-manager-pr02 1/1 Running 0 24m 178.xxx.xxx.xxx pr02 <none> <none>
kube-system kube-flannel-ds-amd64-wwsm7 1/1 Running 0 20m 178.xxx.xxx.xxx pr02 <none> <none>
kube-system kube-proxy-4c9n4 1/1 Running 0 25m 178.xxx.xxx.xxx pr02 <none> <none>
kube-system kube-scheduler-pr02 1/1 Running 0 24m 178.xxx.xxx.xxx pr02 <none> <none>
```


```
kubectl get pods --all-namespaces
kube-system coredns-5644d7b6d9-fl7h8 0/1 Pending 0 6m36s
kube-system coredns-5644d7b6d9-rvh6q 0/1 Pending 0 6m36s
kube-system etcd-pr02 1/1 Running 0 5m31s
kube-system kube-apiserver-pr02 1/1 Running 0 5m40s
kube-system kube-controller-manager-pr02 1/1 Running 0 5m47s
kube-system kube-flannel-ds-amd64-wwsm7 1/1 Running 0 88s
kube-system kube-proxy-4c9n4 1/1 Running 0 6m36s
kube-system kube-scheduler-pr02 1/1 Running 0 5m46s
```


```
kubectl get nodes
NAME STATUS ROLES AGE VERSION
pr02 NotReady master 10m v1.16.0
```


```
kubectl describe pod/coredns-5644d7b6d9-fl7h8 -n kube-system
Name: coredns-5644d7b6d9-fl7h8
Namespace: kube-system
Priority: 2000000000
Priority Class Name: system-cluster-critical
Node:
Labels: k8s-app=kube-dns
pod-template-hash=5644d7b6d9
Annotations:
Status: Pending
IP:
IPs:
Controlled By: ReplicaSet/coredns-5644d7b6d9
Containers:
coredns:
Image: k8s.gcr.io/coredns:1.6.2
Ports: 53/UDP, 53/TCP, 9153/TCP
Host Ports: 0/UDP, 0/TCP, 0/TCP
Args:
-conf
/etc/coredns/Corefile
Limits:
memory: 170Mi
Requests:
cpu: 100m
memory: 70Mi
Liveness: http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
Readiness: http-get http://:8181/ready delay=0s timeout=1s period=10s #success=1 #failure=3
Environment:
Mounts:
/etc/coredns from config-volume (ro)
/var/run/secrets/kubernetes.io/serviceaccount from coredns-token-cww59 (ro)
Conditions:
Type Status
PodScheduled False
Volumes:
config-volume:
Type: ConfigMap (a volume populated by a ConfigMap)
Name: coredns
Optional: false
coredns-token-cww59:
Type: Secret (a volume populated by a Secret)
SecretName: coredns-token-cww59
Optional: false
QoS Class: Burstable
Node-Selectors: beta.kubernetes.io/os=linux
Tolerations: CriticalAddonsOnly
node-role.kubernetes.io/master:NoSchedule
node.kubernetes.io/not-ready:NoExecute for 300s
node.kubernetes.io/unreachable:NoExecute for 300s
Events:
Type Reason Age From Message
Warning FailedScheduling default-scheduler 0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
Warning FailedScheduling default-scheduler 0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
[root@pr02 ~]# kubectl describe pod/coredns-5644d7b6d9-rvh6q -n kube-system
Name: coredns-5644d7b6d9-rvh6q
Namespace: kube-system
Priority: 2000000000
Priority Class Name: system-cluster-critical
Node:
Labels: k8s-app=kube-dns
pod-template-hash=5644d7b6d9
Annotations:
Status: Pending
IP:
IPs:
Controlled By: ReplicaSet/coredns-5644d7b6d9
Containers:
coredns:
Image: k8s.gcr.io/coredns:1.6.2
Ports: 53/UDP, 53/TCP, 9153/TCP
Host Ports: 0/UDP, 0/TCP, 0/TCP
Args:
-conf
/etc/coredns/Corefile
Limits:
memory: 170Mi
Requests:
cpu: 100m
memory: 70Mi
Liveness: http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
Readiness: http-get http://:8181/ready delay=0s timeout=1s period=10s #success=1 #failure=3
Environment:
Mounts:
/etc/coredns from config-volume (ro)
/var/run/secrets/kubernetes.io/serviceaccount from coredns-token-cww59 (ro)
Conditions:
Type Status
PodScheduled False
Volumes:
config-volume:
Type: ConfigMap (a volume populated by a ConfigMap)
Name: coredns
Optional: false
coredns-token-cww59:
Type: Secret (a volume populated by a Secret)
SecretName: coredns-token-cww59
Optional: false
QoS Class: Burstable
Node-Selectors: beta.kubernetes.io/os=linux
Tolerations: CriticalAddonsOnly
node-role.kubernetes.io/master:NoSchedule
node.kubernetes.io/not-ready:NoExecute for 300s
node.kubernetes.io/unreachable:NoExecute for 300s
Events:
Type Reason Age From Message
Warning FailedScheduling default-scheduler 0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
Warning FailedScheduling default-scheduler 0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
```


```
systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
Drop-In: /usr/lib/systemd/system/kubelet.service.d
└─10-kubeadm.conf
Active: active (running) since Вт 2019-09-24 16:32:08 +06; 13min ago
Docs: https://kubernetes.io/docs/
Main PID: 17233 (kubelet)
Tasks: 22
Memory: 42.0M
CGroup: /system.slice/kubelet.service
└─17233 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infr...
сен 24 16:44:59 pr02 kubelet[17233]: W0924 16:44:59.231530 17233 cni.go:237] Unable to update cni config: no valid networks found in /etc/cni/net.d
сен 24 16:45:03 pr02 kubelet[17233]: E0924 16:45:03.072008 17233 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: ...fig uninitialized
сен 24 16:45:04 pr02 kubelet[17233]: W0924 16:45:04.236045 17233 cni.go:202] Error validating CNI config &{cbr0 false [0xc00065f180 0xc00065f200] [123 10 32 32 34 110 97 109 101 34 58 32 34 99 98 114 48 34 44 10...1 112 101 34 58 3
сен 24 16:45:04 pr02 kubelet[17233]: W0924 16:45:04.236124 17233 cni.go:237] Unable to update cni config: no valid networks found in /etc/cni/net.d
сен 24 16:45:08 pr02 kubelet[17233]: E0924 16:45:08.098605 17233 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: ...fig uninitialized
сен 24 16:45:09 pr02 kubelet[17233]: W0924 16:45:09.241163 17233 cni.go:202] Error validating CNI config &{cbr0 false [0xc00049d960 0xc00049da20] [123 10 32 32 34 110 97 109 101 34 58 32 34 99 98 114 48 34 44 10...1 112 101 34 58 3
сен 24 16:45:09 pr02 kubelet[17233]: W0924 16:45:09.241258 17233 cni.go:237] Unable to update cni config: no valid networks found in /etc/cni/net.d
сен 24 16:45:13 pr02 kubelet[17233]: E0924 16:45:13.123498 17233 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: ...fig uninitialized
сен 24 16:45:14 pr02 kubelet[17233]: W0924 16:45:14.246693 17233 cni.go:202] Error validating CNI config &{cbr0 false [0xc0009dc120 0xc0009dc1e0] [123 10 32 32 34 110 97 109 101 34 58 32 34 99 98 114 48 34 44 10...1 112 101 34 58 3
сен 24 16:45:14 pr02 kubelet[17233]: W0924 16:45:14.246781 17233 cni.go:237] Unable to update cni config: no valid networks found in /etc/cni/net.d
Hint: Some lines were ellipsized, use -l to show in full.
```


```
journalctl -f
-- Logs begin at Вт 2019-09-24 16:09:22 +06. --
сен 24 16:45:49 pr02 kubelet[17233]: W0924 16:45:49.283765 17233 cni.go:237] Unable to update cni config: no valid networks found in /etc/cni/net.d
сен 24 16:45:53 pr02 kubelet[17233]: E0924 16:45:53.332983 17233 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
сен 24 16:45:54 pr02 kubelet[17233]: W0924 16:45:54.290296 17233 cni.go:202] Error validating CNI config &{cbr0 false [0xc000a85b20 0xc000a85bc0] [123 10 32 32 34 110 97 109 101 34 58 32 34 99 98 114 48 34 44 10 32 32 34 112 108 117 103 105 110 115 34 58 32 91 10 32 32 32 32 123 10 32 32 32 32 32 32 34 116 121 112 101 34 58 32 34 102 108 97 110 110 101 108 34 44 10 32 32 32 32 32 32 34 100 101 108 101 103 97 116 101 34 58 32 123 10 32 32 32 32 32 32 32 32 34 104 97 105 114 112 105 110 77 111 100 101 34 58 32 116 114 117 101 44 10 32 32 32 32 32 32 32 32 34 105 115 68 101 102 97 117 108 116 71 97 116 101 119 97 121 34 58 32 116 114 117 101 10 32 32 32 32 32 32 125 10 32 32 32 32 125 44 10 32 32 32 32 123 10 32 32 32 32 32 32 34 116 121 112 101 34 58 32 34 112 111 114 116 109 97 112 34 44 10 32 32 32 32 32 32 34 99 97 112 97 98 105 108 105 116 105 101 115 34 58 32 123 10 32 32 32 32 32 32 32 32 34 112 111 114 116 77 97 112 112 105 110 103 115 34 58 32 116 114 117 101 10 32 32 32 32 32 32 125 10 32 32 32 32 125 10 32 32 93 10 125 10]}: [plugin flannel does not support config version ""]
сен 24 16:45:54 pr02 kubelet[17233]: W0924 16:45:54.290397 17233 cni.go:237] Unable to update cni config: no valid networks found in /etc/cni/net.d
сен 24 16:45:58 pr02 kubelet[17233]: E0924 16:45:58.358064 17233 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```


Reactions are currently unavailable