---
title: Docker / containerd Primer
tags: [container, docker, containerd, runc]
---

# Docker / containerd Primer

What sits where in the modern container runtime stack, and which layer
each common failure lives at.

## The stack

```
docker CLI       ─┐
                  │   (Docker API / gRPC)
dockerd          ─┘
   │
   ▼ (CRI on k8s nodes; native on plain Docker hosts)
containerd
   │
   ▼ (OCI runtime spec)
runc  →  Linux kernel (namespaces + cgroups + capabilities + seccomp)
```

Pieces:

- **`docker`** — CLI that talks to dockerd
- **`dockerd`** — the build / image / network / volume daemon
- **`containerd`** — actually runs containers; daemon written by the
  same folks
- **`runc`** — the OCI reference runtime; spawns the container process
  and applies kernel isolation primitives

On a Kubernetes node since k8s 1.24, `dockerd` is **not** in the
picture — kubelet talks to containerd via CRI directly.

## What "isolation" actually means

A container is just a process tree with extra kernel state attached:

| Isolation | Implemented via | Visible failure |
|-----------|----------------|-----------------|
| Process tree | PID namespace | `ps` inside container sees only its own |
| Filesystem | Mount namespace + overlayfs | Image layers, copy-on-write |
| Network | Network namespace + veth pairs | `ip addr` inside container is isolated |
| User IDs | User namespace (optional) | UID inside ≠ UID outside |
| Hostname / IPC | UTS / IPC namespaces | Usually invisible to operators |
| CPU / memory / IO | cgroups v2 | Throttling, OOM |
| Syscalls | seccomp profile | `EPERM` on disallowed syscalls |
| Capabilities | capabilities(7) | "Operation not permitted" despite root |
| MAC | AppArmor / SELinux | Profile-specific denials |

Container "escape" usually means escaping the mount namespace via a
mis-configured `hostPath` or a writable `/proc` from a privileged
container. Privileged containers turn off most of the above; treat
`--privileged` as "this is now a host process".

## Image layers and copy-on-write

Each `RUN` / `COPY` / `ADD` in a Dockerfile creates a layer. Layers are
content-addressed by sha256, stored once on disk, and stacked via
overlayfs at runtime. Implications:

- **Order Dockerfile commands by stability.** Stable steps near the top,
  fast-changing near the bottom — leverages the layer cache for fast
  rebuilds
- **One layer = one decision.** `RUN apt-get update && apt-get install
  ...&& rm -rf /var/lib/apt/lists/*` in a single layer keeps the cache
  cleanup atomic; splitting the cleanup into a later layer leaves the
  cache files in the middle layer forever
- **Layered FS performance ≠ native FS performance.** Heavy random IO
  in a container (e.g. databases) should use a volume mount, not the
  container's layered root

## Volumes vs. bind mounts

- **Volumes** (`/var/lib/docker/volumes/...`) — managed by Docker, can
  be backed by drivers. Default for "I want persistent data".
- **Bind mounts** — direct host path mount. Useful for dev (`-v
  $PWD:/app`) and unavoidable for `/var/run/docker.sock` style.
- **tmpfs** — RAM-backed; good for ephemeral state where you want speed
  and don't want it to survive a restart

## Networking modes

| Mode | Behavior | When |
|------|----------|------|
| bridge (default) | Container on a software bridge with NAT to host | General use |
| host | Container shares host's net namespace | Low-overhead; loses port isolation |
| none | No network at all | Air-gapped jobs |
| user-defined bridge | Custom bridge with built-in DNS for service names | Compose / multi-container apps |
| macvlan | Each container gets a MAC on the physical LAN | When the network team insists on real IPs |

## Common failure modes

| Symptom | Root cause |
|---------|-----------|
| `Cannot connect to the Docker daemon` | dockerd not running, or permission on the socket |
| `pull access denied` | Missing creds or typo'd image name |
| `no space left on device` mid-pull | `/var/lib/docker` full — `docker system prune` |
| Container OOM-killed | `--memory` too low for workload, or RSS leak |
| `exec format error` | Image built for different CPU arch |
| Slow `docker build` | Bad layer ordering; missing `.dockerignore` |
| Container can't reach internet | iptables / firewall rules wiped Docker's NAT chain — restart dockerd to repopulate |

## Useful one-liners

```bash
# Storage usage breakdown
docker system df -v

# Layers of an image
docker history --no-trunc <image>

# What's inside the container
docker exec -it <id> sh
nsenter -t $(docker inspect -f '{{.State.Pid}}' <id>) -a    # full host view

# When the container won't start, look at the exit + logs
docker inspect -f '{{.State.ExitCode}} {{.State.Error}}' <id>
docker logs --tail 200 <id>
```

## How this maps to k8s

Most behaviors above apply equally on a k8s node — the runtime is still
`runc` under containerd. Differences:

- The pause container (one per pod) owns the pod's network namespace;
  app containers join it
- `docker exec` ≈ `kubectl exec`; `docker logs` ≈ `kubectl logs`
- Image pulls go through containerd directly; you debug pulls with
  `crictl pull`, not `docker pull`
