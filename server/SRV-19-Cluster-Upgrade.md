---
tags: [homelab, project, kubernetes, kubeadm, upgrade, etcd, backup]
---

# Cluster Upgrade 1.31 → 1.33

> Status: 🟢 **Done.** The control plane and the worker run Kubernetes 1.33.13; the GPU laptop stays on 1.31.14 on purpose. The upgrade went through two minor versions (1.31 → 1.32 → 1.33), the site stayed reachable except for a few minutes while the worker was drained, and the monitoring settings from [SRV-18](./SRV-18-Backups-and-Platform-Config.md) survived both upgrades. 🟡 Open: the laptop node — see [Open items](#open-items).

Picks up after [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the laptop `<GPU_HOSTNAME>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Rules of the upgrade](#rules-of-the-upgrade)
4. [Step 1 — Test a restore first](#step-1--test-a-restore-first)
5. [Step 2 — Control plane](#step-2--control-plane)
6. [Step 3 — Worker](#step-3--worker)
7. [Step 4 — Repeat for the next minor version](#step-4--repeat-for-the-next-minor-version)
8. [Verification](#verification)
9. [Limits](#limits)
10. [Open items](#open-items)
11. [Related](#related)

## Components

| Component | Before | After |
|---|---|---|
| kubeadm, kubelet, kubectl on `<HOSTNAME>` | 1.31.14 | 1.33.13 |
| kubeadm, kubelet, kubectl on `<WORKER_HOSTNAME>` | 1.31.14 | 1.33.13 |
| kubelet on `<GPU_HOSTNAME>` | 1.31.14 | 1.31.14 (not upgraded) |
| API server, controller manager, scheduler, kube-proxy | 1.31.14 | 1.33.13 |
| etcd, CoreDNS | 3.5.24, 1.11.3 | unchanged |

## Why

Kubernetes 1.31 is past the end of upstream support, so it no longer receives fixes. The next platform component, a Gateway API implementation to replace the retired ingress-nginx, also targets newer releases: Envoy Gateway 1.9 supports Kubernetes 1.33 to 1.36, while the versions that support 1.31 are themselves out of support. Upgrading the cluster first avoids replacing one unsupported component with another.

## Rules of the upgrade

- **One minor version at a time.** `kubeadm` does not skip: 1.31 → 1.32 → 1.33.
- **Control plane first, then nodes.** A kubelet may be older than the API server (up to three minor versions), never newer.
- **Packages are held** (`apt-mark hold`) so a routine `apt upgrade` cannot move the cluster. Each step unholds, installs a pinned version and holds again.
- **A fresh backup before each minor version** ([SRV-18](./SRV-18-Backups-and-Platform-Config.md)).
- **The laptop is left out.** A 1.31 kubelet is valid against a 1.33 control plane, so the node can wait; it must be upgraded before the control plane moves past 1.34.

## Step 1 — Test a restore first

An upgrade without a proven backup relies on an assumption. Restore the newest snapshot into a scratch directory; the running cluster is not touched:

```
which etcdctl || sudo apt install -y etcd-client
LATEST=$(ls -1d /var/backups/homelab/*/ | tail -1)
etcdctl snapshot status ${LATEST}etcd-snapshot.db -w table
etcdctl snapshot restore ${LATEST}etcd-snapshot.db --data-dir ~/restore-test/etcd-data
find ~/restore-test -maxdepth 3
rm -rf ~/restore-test
```

The Ubuntu package `etcd-client` provides `etcdctl` 3.5.16 (the separate `etcdutl` is in `etcd-server`, which also installs an etcd service and is not wanted on the node); `etcdctl` prints a deprecation notice but restores correctly. The status table showed 1374 keys and 28 MB, the restore produced `member/snap` and `member/wal` without errors. The restored directory holds every Secret in clear text, so it is deleted right after the check.

This proves the snapshot is readable and restorable; it does not rehearse a full control-plane rebuild ([Open items](#open-items)).

## Step 2 — Control plane

Take a backup, point the package repository at the next minor version and install a matching `kubeadm`:

```
sudo ~/bin/backup-homelab.sh 2>&1 | tail -1
sudo sed -i 's|/v1.31/|/v1.32/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt update
apt-cache madison kubeadm | head -3
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.32.13-1.1'
sudo apt-mark hold kubeadm
kubeadm version -o short
sudo kubeadm upgrade plan
```

`upgrade plan` reads the cluster, lists what will change and names the exact `apply` command. Anything marked `[ERROR]` stops here. In this cluster it listed only the expected components and no manual config upgrades. Then apply:

```
sudo kubeadm upgrade apply v1.32.13 -y
```

`apply` rewrites the static pod manifests (etcd, API server, controller manager, scheduler), renews their certificates, applies CoreDNS and kube-proxy, and ends with `SUCCESS`. The API server is unreachable for a minute or two; running pods are not affected.

Check that the metrics settings from [SRV-18 Step 7](./SRV-18-Backups-and-Platform-Config.md#step-7--make-control-plane-metrics-survive-upgrades) were carried over:

```
sudo grep -E 'bind-address|listen-metrics' /etc/kubernetes/manifests/*.yaml
kubectl -n kube-system get cm kube-proxy -o yaml | grep metricsBindAddress
```

The etcd `--listen-metrics-urls`, both `--bind-address=0.0.0.0` flags and `metricsBindAddress: 0.0.0.0:10249` were all still there, after both upgrades. The new manifests are generated from the `kubeadm-config` ConfigMap, so this is the proof that editing the ConfigMap instead of the manifests was the right place.

Then upgrade the node's own kubelet and kubectl. Draining moves the pods that are not DaemonSet-managed (CoreDNS) to the worker:

```
kubectl drain <HOSTNAME> --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.32.13-1.1' kubectl='1.32.13-1.1'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon <HOSTNAME>
kubectl get nodes
```

The control-plane node shows the new version. The API server may refuse connections for about a minute after the kubelet restart, because the new kubelet recreates the static pods ([troubleshooting](../troubleshooting/SRV-19-TRBL-Cluster-Upgrade.md)).

## Step 3 — Worker

The worker is the only node that accepts ordinary pods (see [SRV-17](./SRV-17-Dedicated-GPU-Node.md#node-roles)), so draining it stops every application — the site, Grafana, ArgoCD, Loki, Prometheus, Alertmanager and cluster DNS — for a few minutes. Monitoring is down together with them, so no alerts fire during the outage; one or two may appear when it comes back. Pods stay `Pending` until the node is uncordoned.

On the worker, prepare the node while everything still runs:

```
sudo sed -i 's|/v1.31/|/v1.32/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.32.13-1.1'
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
```

On the control plane, drain the worker (the outage begins):

```
kubectl drain <WORKER_HOSTNAME> --ignore-daemonsets --delete-emptydir-data
```

`--delete-emptydir-data` is needed because Grafana and Prometheus pods use `emptyDir` volumes; their persistent data lives in the `local-path` volumes on the same node and is not touched.

On the worker, upgrade and restart the kubelet:

```
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.32.13-1.1' kubectl='1.32.13-1.1'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

On the control plane, bring the node back:

```
kubectl uncordon <WORKER_HOSTNAME>
kubectl get nodes
```

`kubeadm upgrade node` on a worker only updates the local kubelet configuration, which is why it needs no admin credentials; `kubeadm upgrade apply` exists only on the control plane.

## Step 4 — Repeat for the next minor version

The same two steps with `v1.33` in the repository line. Since the exact patch version is the newest in the repository, `kubeadm='1.33.*'` is enough for the install, and `kubeadm version -o short` then gives the version for `upgrade apply` (`v1.33.13`). A backup is taken again before it.

## Verification

After each minor version, once the worker's pods are back (3–5 minutes):

```
kubectl get nodes
kubectl get pods -A | grep -v -E 'Running|Completed'
curl -s -o /dev/null -w "site: %{http_code}\n" http://<LB_IP>/site/
kubectl get --raw '/api/v1/namespaces/monitoring/services/prometheus-kube-prometheus-prometheus:9090/proxy/api/v1/targets?state=active' | python3 -c 'import sys,json;t=json.load(sys.stdin)["data"]["activeTargets"];print(len(t),[x["scrapePool"] for x in t if x["health"]!="up"])'
kubectl -n argocd get applications
```

Nodes on the new version and `Ready`, no pods outside `Running`, the site answering `200`, `28 []` for the scrape targets and `my-site` `Synced`/`Healthy`. Right after the worker returns the target check can list a few `down` targets (API server, CoreDNS, kubelet, node-exporter) for a minute or two; they recover without action, and a second query shows none.

## Limits

- **One worker means an outage.** The upgrade of the worker cannot be rolling while it is the only schedulable node. A second schedulable node would make it a rolling upgrade.
- **A skipped node.** The laptop's kubelet is two minor versions behind the control plane, which is allowed (up to three). That is acceptable for now and must be fixed before the control plane moves to 1.34.
- **No rollback was needed or rehearsed.** The safety net was the backup (Step 1 proves the snapshot restores); kubeadm also keeps the previous static pod manifests in `/etc/kubernetes/tmp/` for each upgrade run.
- **Packages stay held.** Security updates for the OS arrive with `apt upgrade`, but the Kubernetes packages move only through this procedure.

## Open items

- ⬜ **Upgrade `<GPU_HOSTNAME>`** to 1.33 (same steps as the worker; the AI node taint and workloads need no changes), at the latest before the control plane moves to 1.34.
- ⬜ **Rehearse a full control-plane rebuild** from a backup on a scratch machine, beyond restoring the snapshot.
- ⬜ **Sandbox image warning:** kubeadm reports that the container runtime's pause image differs from the one it expects (`registry.k8s.io/pause:3.10`). It is harmless today; align the containerd config at some point.

## Related

- [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md) — the backup this relies on and the `kubeadm-config` settings it validated
- [Dedicated GPU Node](./SRV-17-Dedicated-GPU-Node.md) — why the worker is the only node for ordinary pods
- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — the control-plane metrics that had to survive the upgrade
- [SRV-19-TRBL](../troubleshooting/SRV-19-TRBL-Cluster-Upgrade.md) — problems hit during this step
