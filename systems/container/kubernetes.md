---
title: Kubernetes Primer
tags: [kubernetes, k8s, container, orchestration]
---

# Kubernetes Primer

The model in one diagram, then the components that implement it, then
the failure modes you actually meet in production.

## The control loop model

Everything in k8s is a control loop over **desired state vs. observed
state**, both stored in etcd. Controllers (one per object kind) watch
their slice of etcd, compare desired to observed, and emit changes to
move observed closer to desired. If you understand that one sentence,
most of k8s behavior is predictable.

```
                       ┌────────────────────┐
       apply manifest  │                    │  changes
       ──────────────▶ │       etcd         │ ◀──────────── controllers
                       │  (desired + obs)   │
                       └─────────┬──────────┘
                                 │ watch
                                 ▼
                   ┌──────────────────────────┐
                   │  kube-apiserver          │
                   └──────────────────────────┘
                       ▲                    ▲
                       │                    │
                   kubelet              scheduler / controllers
                  (per node)
```

## Components

### Control plane

- **kube-apiserver** — REST front door; only thing that talks to etcd
- **etcd** — strongly-consistent KV store; the cluster's only state
- **kube-scheduler** — places pending pods on nodes (see
  `systems/scheduling/kubernetes-scheduler.md`)
- **kube-controller-manager** — runs the built-in controllers
  (deployment, replicaset, node, endpoints, ...)
- **cloud-controller-manager** — IaaS-specific bits (LB, route, volume
  provisioning)

### Data plane (each node)

- **kubelet** — agent that turns pod specs into running containers via
  the CRI
- **kube-proxy** — implements Service IP → backend pod routing
  (iptables / ipvs / nftables)
- **CNI plugin** — sets up pod networking (Cilium, Calico, Flannel, ...)
- **CRI runtime** — containerd / cri-o; talks to runc

## Object hierarchy

```
Deployment  →  ReplicaSet  →  Pod  →  Container
StatefulSet →                Pod  →  Container
DaemonSet   →                Pod  →  Container
Job         →                Pod  →  Container
```

Higher-level controllers manage lower-level ones; manual edits to lower
levels get reverted by the higher controller. Edit at the right level.

## Pod lifecycle

```
Pending → ContainerCreating → Running → (Succeeded | Failed | Terminating)
```

The transitions that produce on-call pages:

- **Pending stuck** — see scheduler doc; usually filter-phase failure
- **ImagePullBackOff** — registry creds, image typo, network to registry
- **CrashLoopBackOff** — container exits non-zero; backoff doubles each
  retry up to 5 min
- **Terminating stuck** — preStop hook hung, or finalizer not removed
  (force delete with `--grace-period=0 --force` only if you understand
  the finalizer)

## Probes — what they actually do

- **liveness** — if it fails, kubelet *restarts* the container
- **readiness** — if it fails, Service stops routing traffic to this pod
- **startup** — disables liveness until startup passes (use for slow
  starters; otherwise liveness may kill a still-booting container)

**Pitfall.** A liveness probe that hits the same endpoint as readiness,
with the same code path, will turn a temporary DB-connection drop into
a restart loop. Liveness should ask "is the *process* hung", not "are
its deps healthy". When in doubt, omit liveness.

## Networking layers

```
Container ports
   ↓ (CNI)
Pod IP  (unique cluster-wide)
   ↓ (kube-proxy or eBPF datapath)
Service IP  (cluster virtual IP, load-balanced across endpoints)
   ↓ (Ingress / LB)
External
```

- **ClusterIP** — internal only, default
- **NodePort** — same plus a port opened on every node (avoid in
  production; use it for dev)
- **LoadBalancer** — cloud-provisioned LB sits in front
- **Ingress** — L7 routing via an ingress controller (nginx, traefik,
  envoy); requires a controller installed in-cluster

## Storage layers

- **emptyDir** — node-local scratch; lost on pod restart
- **hostPath** — node directory mount; ties pod to a host
- **PersistentVolume + PersistentVolumeClaim** — abstract storage; PV
  provisioned (statically or dynamically via a StorageClass), PVC binds
  to it
- **CSI driver** — the plug-in interface between k8s and your storage
  backend; if your cloud provider doesn't ship one, the storage doesn't
  work

## RBAC in one paragraph

`Role` and `ClusterRole` grant verbs on resources. `RoleBinding` and
`ClusterRoleBinding` attach a Role to a subject (user / group / service
account). Service accounts are the *normal* identity for in-cluster
processes; mounting their token (`automountServiceAccountToken: true`)
is the default and almost always more privilege than the workload
needs — set it to `false` unless the pod must talk to the API.

## Where AIOps care points up

For an AIOps platform watching k8s, the high-value signals are:

- **Pod restart count** + last-exit reason
- **Pending-pod age** (filter-phase failures don't self-heal)
- **Node condition** transitions (Ready ↔ NotReady)
- **PV bound→released→failed** transitions
- **kube-apiserver request latency p99** by verb/resource (early
  indicator of etcd or controller storms)
- **Number of objects per kind** (controller storms manifest as endless
  re-list/relist)
