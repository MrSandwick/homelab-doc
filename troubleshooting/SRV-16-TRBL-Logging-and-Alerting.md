---
tags: [homelab, project, kubernetes, logging, loki, alloy, alertmanager, slack, troubleshooting]
---

# Troubleshooting — Logging and Alerting

> Companion to [Logging and Alerting](../server/SRV-16-Logging-and-Alerting.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [Alloy warnings: `no route to host` for the laptop node](#alloy-warnings-no-route-to-host-for-the-laptop-node) | [Step 3](../server/SRV-16-Logging-and-Alerting.md#step-3--keep-the-laptop-node-out-of-log-collection) |
| [Slack: no "From scratch" option](#slack-no-from-scratch-option) | [Step 5](../server/SRV-16-Logging-and-Alerting.md#step-5--slack-webhook-and-secret) |

## Alloy warnings: `no route to host` for the laptop node

**Symptom:** right after the first install, Alloy's log was full of lines like:

```
level=warn msg="tailer stopped; will retry" target=kube-system/kube-proxy-...:kube-proxy
err="Get \"https://<GPU_NODE_IP>:10250/containerLogs/...\": dial tcp <GPU_NODE_IP>:10250: connect: no route to host"
```

for kube-proxy, Flannel, the MetalLB speaker and node-exporter pods.

**Root cause:** all of those pods live on the laptop node. The node is `NotReady` (its owner shuts it down), but its system DaemonSet pods are still registered, so Alloy discovered them and tried to reach that node's kubelet. They exist at all because `kubectl cordon` only blocks ordinary scheduling: DaemonSet pods tolerate the unschedulable taint and run on a cordoned node anyway. The cordon decision ([SRV-13](../server/SRV-13-ArgoCD-GitOps.md#open-items)) therefore keeps our services off the laptop, but not the cluster's per-node system pods, and that is normal for a node that is part of the cluster.

This was also why Alloy was installed as a one-replica Deployment: as a DaemonSet it would have added one more pod of ours to that node.

**Fix:** a `drop` rule on the node name in the Alloy pipeline ([Step 3](../server/SRV-16-Logging-and-Alerting.md#step-3--keep-the-laptop-node-out-of-log-collection)), and a matching pair of `"null"` routes in Alertmanager so alerts about that node are not sent to Slack ([Step 6](../server/SRV-16-Logging-and-Alerting.md#step-6--alertmanager-routing)). After the rollout, counting `no route to host` in the Alloy log gave `0`.

**Not covered:** alerts that carry no `node` or `instance` label of that node. Checking this with the laptop offline is an open item.

## Slack: no "From scratch" option

**Symptom:** the step "Create New App → From scratch" did not match the screen. The dialog offered *AI agent*, *Starter app*, *From a manifest* and *Blank app*.

**Root cause:** Slack renamed the entry; the screen is a newer version of the dialog than the instructions assumed.

**Fix:** **Blank app** is the same option (an empty app with minimal setup). After it, the steps are unchanged: **Incoming Webhooks** → activate → **Add New Webhook to Workspace**.
