---
tags: [homelab, project, kubernetes, scheduling, taints, daemonset, troubleshooting]
---

# Troubleshooting — Dedicated GPU Node

> Companion to [Dedicated GPU Node](../server/SRV-17-Dedicated-GPU-Node.md), which records only the working procedure. This file records the problems hit during that work.

## Contents

| Incident | Step |
|---|---|
| [DaemonSet alerts after tainting the node](#daemonset-alerts-after-tainting-the-node) | [Step 5](../server/SRV-17-Dedicated-GPU-Node.md#step-5--daemonsets-and-the-taint) |

## DaemonSet alerts after tainting the node

**Symptom:** a few minutes after the node was tainted and uncordoned, Slack received:

```
[FIRING:2] KubeDaemonSetMisScheduled
1 Pods of DaemonSet metallb-system/metallb-frr-k8s are running where they are not supposed to run
1 Pods of DaemonSet metallb-system/metallb-speaker are running where they are not supposed to run
[FIRING:1] KubeDaemonSetMisScheduled   (kube-system/nvidia-device-plugin-daemonset)
[FIRING:2] KubeDaemonSetRolloutStuck   (metallb-frr-k8s, metallb-speaker, after 15 minutes)
[FIRING:1] KubeDaemonSetRolloutStuck   (nvidia-device-plugin-daemonset)
```

Nothing was broken yet: the pods were running.

**Root cause:** a DaemonSet controller adds tolerations for a few built-in taints (unschedulable, not ready) automatically, which is why these pods ran on the cordoned node. A custom taint is not covered. Flannel, kube-proxy and node-exporter carry a blanket toleration, but the Device Plugin and MetalLB had none for `dedicated=ai`, so for them the node was no longer eligible. `NoSchedule` does not evict pods that already run, so they stayed and were counted as misscheduled; the rollout alert follows from the same mismatch.

The real risk was in the Device Plugin: a recreated pod would not have been placed on the laptop again, and the node's GPU would have vanished from the cluster. The alert rules from the monitoring stack caught this the first time it mattered.

**Fix:** a toleration for `dedicated=ai` on the Device Plugin DaemonSet, and removal of the MetalLB pods from the laptop, which should not run there anyway ([Step 5](../server/SRV-17-Dedicated-GPU-Node.md#step-5--daemonsets-and-the-taint)). The GPU stayed allocatable throughout (`nvidia.com/gpu: 1`).

**Lesson:** before tainting a node, list the DaemonSets that run on it and their tolerations, and decide for each whether it must stay:

```
kubectl get ds -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,TOLERATIONS:.spec.template.spec.tolerations
```

The earlier cordon-only setup never needed this step, because a cordon's taint is one the DaemonSet controller tolerates.
