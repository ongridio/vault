---
title: Kubernetes ImagePullBackOff / ErrImagePull
kind: howto
tags: [kubernetes, image, registry, imagepullbackoff, pull-secret, containers]
applies_to: [edge, manager]
---

# Kubernetes ImagePullBackOff / ErrImagePull

Use when a pod is stuck `ImagePullBackOff` / `ErrImagePull`. The kubelet
couldn't fetch the image; the `Events` on the pod almost always name the
exact reason. **Read the event message first** — it distinguishes
not-found, auth, and network in one line.

| Event message contains | Probable cause |
|---|---|
| `manifest unknown` / `not found` | wrong image name or tag (typo, missing tag) |
| `unauthorized` / `denied` | missing/invalid imagePullSecret or registry auth |
| `dial tcp ... timeout` / `no such host` | node can't reach the registry (network/DNS) |
| `toomanyrequests` | registry rate-limit (e.g. Docker Hub anonymous pulls) |

## Step 1 — Read the exact failure

```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
#   look for: Failed to pull image "...": <the real error>
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'   # what it's trying to pull
```

The `Failed to pull image` event carries the registry's own error
verbatim — that string is the diagnosis.

## Step 2 — Name/tag vs auth vs network

```bash
# From a node (or a debug pod on the same node), try the pull manually:
crictl pull <image>            # containerd nodes
#   or: docker pull <image>    # docker nodes
# Auth path — is the pull secret present + correct?
kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets[*].name}{"\n"}'
kubectl get secret <pull-secret> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

- `not found` → fix the image/tag (check it exists in the registry).
- `unauthorized` → the pull secret is missing, wrong, or not referenced
  by the pod's serviceAccount.
- timeout/DNS → node→registry network or DNS (see Step 3).

## Step 3 — Node → registry reachability

```bash
# On the node: can it resolve + reach the registry?
nslookup <registry-host>
curl -fsS -o /dev/null -w '%{http_code}\n' https://<registry-host>/v2/
```

A private/air-gapped registry behind a firewall, a proxy the kubelet
doesn't use, or broken cluster DNS all surface here. (Cross-reference
`diagnostics/dns-resolution-failure.md`.)

## Step 4 — Rate limits & policy

`toomanyrequests` from Docker Hub = anonymous/free pull limit — add
authenticated pull credentials or mirror the image. Also check
`imagePullPolicy: Always` forcing a pull every start when the tag is
mutable; pin digests for stability.

## Decision tree

| Signal | Action |
|---|---|
| `manifest unknown` / not found | fix image name/tag; confirm it exists in registry |
| `unauthorized` / `denied` | create/fix imagePullSecret; attach to SA/pod |
| timeout / `no such host` | node→registry network/DNS — fix routing/proxy/DNS |
| `toomanyrequests` | authenticate pulls or mirror image; pin digest |
| works via crictl on node, not in pod | pull secret not referenced by the pod's SA |

## References

- [Kubernetes — Images & imagePullSecrets](https://kubernetes.io/docs/concepts/containers/images/)
- [Debug Pods — Kubernetes docs](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- vault: `diagnostics/dns-resolution-failure.md`, `diagnostics/k8s-pod-stuck.md`, `systems/container/kubernetes.md`
