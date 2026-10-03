---
tags: [homelab, project, kubernetes, workload, scheduling, troubleshooting]
---

# Troubleshooting — First Real Workload

> Companion to [First Real Workload](../server/SRV-10-First-Real-Workload.md), which records only the working procedure. This file records the problem hit during that work.

## First deployment pinned to the control plane (reverted)

**Relates to:** [Step 1](../server/SRV-10-First-Real-Workload.md#step-1--build-the-image) – [Step 3](../server/SRV-10-First-Real-Workload.md#step-3--deployment-on-the-worker).

**What happened:** the image was first built and imported into containerd on the control-plane node (`<HOSTNAME>`):

```
cd ~/my-site
docker build -t my-site:v1 .
docker save my-site:v1 | sudo ctr -n k8s.io images import -
```

Because the image existed only there, the first Deployment was pinned to that node:

```yaml
nodeSelector:
  kubernetes.io/hostname: <HOSTNAME>
```

The pod stayed `Pending`:

```
0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }
```

A toleration was added and the pod ran:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

**Root cause:** node placement followed where the image had been imported, not where the workload belonged. The result put an application on the control plane, contrary to the taint restored in [Kubernetes Installation, Step 10](../server/SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint).

**Resolution:** everything deployed so far was removed:

```
kubectl delete -f deployment.yaml
kubectl delete -f ingress.yaml
```

The site source was then copied to the worker, the image rebuilt and imported there, and the Deployment re-applied with `nodeSelector` pointing at the worker and the `tolerations` block removed — the procedure recorded in [First Real Workload](../server/SRV-10-First-Real-Workload.md).

## Related

- [First Real Workload](../server/SRV-10-First-Real-Workload.md) — the working procedure
- [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Local-Container-Images-Without-a-Registry.md) — companion Guides repository
- [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
