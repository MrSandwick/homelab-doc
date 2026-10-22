---
tags: [homelab, project, kubernetes, kubeadm, upgrade, troubleshooting]
---

# Troubleshooting — Cluster Upgrade

> Companion to [Cluster Upgrade 1.31 → 1.33](../server/SRV-19-Cluster-Upgrade.md), which records only the working procedure. This file records the problems hit during that work.

## Contents

| Incident | Step |
|---|---|
| [`kubeadm upgrade apply` run on the worker](#kubeadm-upgrade-apply-run-on-the-worker) | [Step 2](../server/SRV-19-Cluster-Upgrade.md#step-2--control-plane) |
| [API server refuses connections after the kubelet restart](#api-server-refuses-connections-after-the-kubelet-restart) | [Step 2](../server/SRV-19-Cluster-Upgrade.md#step-2--control-plane) |
| [Scrape targets `down` after the worker returns](#scrape-targets-down-after-the-worker-returns) | [Step 3](../server/SRV-19-Cluster-Upgrade.md#step-3--worker) |

## `kubeadm upgrade apply` run on the worker

**Symptom:**

```
couldn't create a Kubernetes client from file "/etc/kubernetes/admin.conf": failed to load admin kubeconfig: open /etc/kubernetes/admin.conf: no such file or directory
```

**Root cause:** the command was typed in the worker's terminal. `upgrade apply` upgrades the control plane and needs `/etc/kubernetes/admin.conf`, which exists only on the control-plane node. Nothing was changed.

**Fix:** run it on the control plane. Every command block in the procedure starts with `hostname` for this reason: with several terminals open, the prompt is easy to misread.

## API server refuses connections after the kubelet restart

**Symptom:** right after `systemctl restart kubelet` on the control plane, `kubectl` answers `connection refused` on port 6443 for about a minute. The kubelet log shows `connection refused` and then a few `forbidden: ... no relationship found between node ... and this object`.

**Root cause:** the new kubelet version computes a different hash for the static pod specs and recreates the etcd and API server containers (`crictl ps -a` shows the old ones `Exited` and new ones `Running` with a higher attempt number). While the API server restarts, the kubelet cannot reach it; the `forbidden` lines appear in the short window after it is back and before the node authorizer has the node's pod relationships again.

**Fix:** none needed; wait one to two minutes. It is a pause in the management API only, running workloads are unaffected. To confirm it is only a restart, `sudo crictl ps -a | grep -E 'apiserver|etcd'` shows both `Running`. Do not continue to the worker until `kubectl get nodes` answers.

## Scrape targets `down` after the worker returns

**Symptom:** the target check printed five failing scrape pools (API server, CoreDNS, kubelet twice, node-exporter) a minute after the worker was uncordoned.

**Root cause:** Prometheus, CoreDNS and the monitoring pods had just been recreated on the worker, and the first scrapes ran before the targets were ready. The next check, a minute later, listed none.

**Fix:** none; repeat the check after a few minutes before looking for a real cause.
