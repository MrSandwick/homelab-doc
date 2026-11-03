---
tags: [homelab, project, kubernetes, kubeadm, upgrade, etcd, backup]
---

# Cluster Upgrade 1.31 → 1.33

> Status: 🟢 **Done.** All three nodes run Kubernetes 1.33.13. The control plane and the worker were upgraded by hand; the GPU laptop followed later with an Ansible playbook that applies the same procedure to any node that needs it ([Step 5](#step-5--laptop-node-with-ansible)). The upgrade went through two minor versions (1.31 → 1.32 → 1.33), the site stayed reachable except for a few minutes while the worker was drained, and the monitoring settings from [SRV-18](./SRV-18-Backups-and-Platform-Config.md) survived both upgrades. 🟡 Open: a full control-plane rebuild has not been rehearsed — see [Open items](#open-items).

Picks up after [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the laptop `<GPU_HOSTNAME>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Rules of the upgrade](#rules-of-the-upgrade)
4. [Step 1 — Test a restore first](#step-1--test-a-restore-first)
5. [Step 2 — Control plane](#step-2--control-plane)
6. [Step 3 — Worker](#step-3--worker)
7. [Step 4 — Repeat for the next minor version](#step-4--repeat-for-the-next-minor-version)
8. [Step 5 — Laptop node with Ansible](#step-5--laptop-node-with-ansible)
9. [Verification](#verification)
10. [Limits](#limits)
11. [Open items](#open-items)
12. [Related](#related)

## Components

| Component | Before | After |
|---|---|---|
| kubeadm, kubelet, kubectl on `<HOSTNAME>` | 1.31.14 | 1.33.13 |
| kubeadm, kubelet, kubectl on `<WORKER_HOSTNAME>` | 1.31.14 | 1.33.13 |
| kubeadm, kubelet, kubectl on `<GPU_HOSTNAME>` | 1.31.14 | 1.33.13 (later, with Ansible — Step 5) |
| API server, controller manager, scheduler, kube-proxy | 1.31.14 | 1.33.13 |
| etcd, CoreDNS | 3.5.24, 1.11.3 | unchanged |

## Why

Kubernetes 1.31 is past the end of upstream support, so it no longer receives fixes. The next platform component, a Gateway API implementation to replace the retired ingress-nginx, also targets newer releases: Envoy Gateway 1.9 supports Kubernetes 1.33 to 1.36, while the versions that support 1.31 are themselves out of support. Upgrading the cluster first avoids replacing one unsupported component with another.

## Rules of the upgrade

- **One minor version at a time.** `kubeadm` does not skip: 1.31 → 1.32 → 1.33.
- **Control plane first, then nodes.** A kubelet may be older than the API server (up to three minor versions), never newer.
- **Packages are held** (`apt-mark hold`) so a routine `apt upgrade` cannot move the cluster. Each step unholds, installs a pinned version and holds again.
- **A fresh backup before each minor version** ([SRV-18](./SRV-18-Backups-and-Platform-Config.md)).
- **The laptop is left out at first.** A 1.31 kubelet is valid against a 1.33 control plane, so the node could wait; it was upgraded later with Ansible ([Step 5](#step-5--laptop-node-with-ansible)).

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

## Step 5 — Laptop node with Ansible

The laptop was upgraded later, when the warning `KubeVersionMismatch` ("2 different semantic versions of Kubernetes components") arrived in Slack. Instead of repeating Step 3 by hand twice, the procedure became an Ansible playbook, `upgrade-k8s.yml`, that does Step 2 or Step 3 for whichever node needs it.

**Inventory.** The laptop had been kept out of Ansible ([SRV-17](./SRV-17-Dedicated-GPU-Node.md)). It now has its own group, so `site.yml`, which targets `k8s_cluster`, still never touches it; cluster-wide plays use `k8s_all`:

```
[k8s_gpu]
<GPU_HOSTNAME> ansible_host=<GPU_IP> ansible_user=<GPU_USER>

[k8s_all:children]
k8s_cluster
k8s_gpu
```

The laptop's address had changed since its first setup, so master needed its host key once (`ssh-keyscan -H <GPU_IP> >> ~/.ssh/known_hosts`); the SSH key was already authorized. `ansible -i inventory.ini k8s_all -m ping` answered `pong` for all three nodes.

**Playbook.** One run moves every node that needs it by exactly one minor version:

| Phase | Tasks | Runs on |
|---|---|---|
| Decide | Read the node's kubelet version from the API; skip the node (`meta: end_host`) if it is already on the target; refuse if the target is more than one minor ahead | Control plane (`delegate_to`) |
| Packages | Point `/etc/apt/sources.list.d/kubernetes.list` at `v<minor>`, find the newest patch with `apt-cache madison`, upgrade `kubeadm` (unhold → install → hold) | The node |
| kubeadm | `kubeadm upgrade apply -y v<version>` on the control plane, `kubeadm upgrade node` on workers | The node |
| kubelet | Drain; upgrade `kubelet` and `kubectl` (unhold → install → hold); restart kubelet; wait for `Ready`; uncordon | Drain, wait and uncordon on the control plane, the rest on the node |

`serial: 1` and the inventory order (control plane, worker, laptop) give the order the kubeadm documentation requires: control plane first, then workers one at a time. `gather_facts: false` and `become` only on the tasks that need root mean that skipped nodes never ask for a sudo password, so `-K` takes only the laptop's.

```
ansible-playbook -i inventory.ini upgrade-k8s.yml --syntax-check
ansible-playbook -i inventory.ini upgrade-k8s.yml -e k8s_minor=1.32 -K
ansible-playbook -i inventory.ini upgrade-k8s.yml -e k8s_minor=1.33 -K
kubectl get nodes
```

Both runs: control plane and worker `ok=3 changed=0` (skipped after the version check), the laptop `ok=16 changed=7 failed=0`. All three nodes report `v1.33.13`; the laptop keeps its taint `dedicated=ai:NoSchedule` and label `workload=ai` (drain and uncordon do not change taints). `KubeVersionMismatch` resolved a few minutes later.

`group_vars/all.yml` still pinned `kubernetes_apt_version: "v1.31"`, which a later `site.yml` run would have written back into the repository line of the control plane and the worker; it is now `v1.33`.

The Ansible directory (`~/ansible-homelab` on the control plane) was not under version control. It is copied into `homelab-gitops` as `ops/ansible/` (inventory, `site.yml`, `upgrade-k8s.yml`, `group_vars`, `host_vars`); after the merge, `~/ansible-homelab` becomes a symlink to it. The next upgrade of the whole cluster is one command per minor version, for example `-e k8s_minor=1.34`.

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
- **Ansible runs only what kubeadm allows.** The playbook moves one minor version per run and refuses to skip one; a node that is two versions behind needs two runs.
- **One sudo password per run.** `-K` asks once; nodes with different sudo passwords need separate runs with `--limit`, or the passwords in Ansible Vault.
- **No rollback was needed or rehearsed.** The safety net was the backup (Step 1 proves the snapshot restores); kubeadm also keeps the previous static pod manifests in `/etc/kubernetes/tmp/` for each upgrade run.
- **Packages stay held.** Security updates for the OS arrive with `apt upgrade`, but the Kubernetes packages move only through this procedure.

## Open items

- ✅ **Upgrade `<GPU_HOSTNAME>`** to 1.33 — done with Ansible ([Step 5](#step-5--laptop-node-with-ansible)).
- ⬜ **Rehearse a full control-plane rebuild** from a backup on a scratch machine, beyond restoring the snapshot.
- ⬜ **Sandbox image warning:** kubeadm reports that the container runtime's pause image differs from the one it expects (`registry.k8s.io/pause:3.10`). It is harmless today; align the containerd config at some point.

## Related

- [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md) — the backup this relies on and the `kubeadm-config` settings it validated
- [Dedicated GPU Node](./SRV-17-Dedicated-GPU-Node.md) — why the worker is the only node for ordinary pods
- [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md) — the Ansible setup the upgrade playbook builds on
- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — the control-plane metrics that had to survive the upgrade
- [SRV-19-TRBL](../troubleshooting/SRV-19-TRBL-Cluster-Upgrade.md) — problems hit during this step
