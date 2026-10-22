---
tags: [homelab, project, kubernetes, backup, etcd, cron, gitops, alertmanager, kubeadm]
---

# Backups and Platform Config

> Status: 🟢 **Working.** A nightly job saves the cluster state (etcd, certificates, Helm values, Grafana data) on the control-plane node and mirrors it to the worker; seven nightly runs were checked on both nodes. The Helm values live in Git, Alertmanager has a persistent volume, and the control-plane metrics settings survive `kubeadm upgrade`. 🟡 Open: no restore test, no off-site copy — see [Open items](#open-items).

Picks up after [Dedicated GPU Node](./SRV-17-Dedicated-GPU-Node.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>` (`<WORKER_IP>`), the admin user `<USER>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Step 1 — The backup script](#step-1--the-backup-script)
4. [Step 2 — Run it once by hand](#step-2--run-it-once-by-hand)
5. [Step 3 — Copy to the worker](#step-3--copy-to-the-worker)
6. [Step 4 — Schedule both](#step-4--schedule-both)
7. [Step 5 — Helm values into Git](#step-5--helm-values-into-git)
8. [Step 6 — A volume for Alertmanager](#step-6--a-volume-for-alertmanager)
9. [Step 7 — Make control-plane metrics survive upgrades](#step-7--make-control-plane-metrics-survive-upgrades)
10. [Verification](#verification)
11. [Limits](#limits)
12. [Open items](#open-items)
13. [Files](#files)
14. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| `backup-homelab.sh` | `~/bin/` on `<HOSTNAME>` | Creates one timestamped backup, keeps the last 7 |
| `/var/backups/homelab/<timestamp>/` | `<HOSTNAME>` | `etcd-snapshot.db`, `pki.tar.gz`, `k8s-manifests.tar.gz`, `grafana-data.tar.gz` |
| `/etc/cron.d/homelab-backup` | `<HOSTNAME>` | Runs the script as root at 03:15 (server time) |
| `rsync` job in `<USER>`'s crontab | `<HOSTNAME>` | Mirrors the backup directory to the worker at 03:30 |
| `~/backups-from-master/` | `<WORKER_HOSTNAME>` | The second copy, on another machine |
| `platform/` | `homelab-gitops` repository | Helm values and config of every platform release |

## Why

The cluster had persistent volumes ([SRV-15](./SRV-15-Persistent-Storage.md)) but nothing protected them, and the whole cluster state lived on one disk. What matters, in order:

- **etcd** holds every Kubernetes object. Losing it means rebuilding the cluster by hand.
- **Certificates** (`/etc/kubernetes/pki`) are needed to bring a control plane back.
- **Helm values and manifests** are small but are the only record of how the platform was installed.
- **Grafana data** (dashboards, users) is small but hand-made. Prometheus and Loki data is deliberately not backed up: metrics and logs are regenerated, and their volumes are large.

A copy on the same disk does not survive a disk failure, so the job also mirrors it to the worker.

## Step 1 — The backup script

Create `~/bin/backup-homelab.sh` (replace the placeholders):

```
#!/usr/bin/env bash
# Backs up cluster state to /var/backups/homelab on the control plane.
# Run as root (needs /etc/kubernetes). Keeps the last 7 backups.
set -euo pipefail
umask 077
export KUBECONFIG=/etc/kubernetes/admin.conf
DEST=/var/backups/homelab
OUT="$DEST/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUT"
chmod 700 "$DEST" "$OUT"
# always hand the files back to the admin user, even if a step fails
trap 'chown -R <USER>:<USER> "$DEST"' EXIT

# 1. etcd snapshot (contains every Secret in clear text: keep the backup private)
kubectl -n kube-system exec etcd-<HOSTNAME> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/backup-snapshot.db
mv /var/lib/etcd/backup-snapshot.db "$OUT/etcd-snapshot.db"

# 2. control-plane certificates
tar czf "$OUT/pki.tar.gz" -C /etc/kubernetes pki

# 3. Helm values and manifests
tar czf "$OUT/k8s-manifests.tar.gz" -C /home/<USER> k8s-manifests

# 4. Grafana data: a short-lived pod mounts the volume read-only (the Grafana image has no tar)
kubectl -n monitoring delete pod grafana-backup --ignore-not-found
kubectl -n monitoring apply -f - <<'POD'
apiVersion: v1
kind: Pod
metadata:
  name: grafana-backup
  namespace: monitoring
spec:
  restartPolicy: Never
  containers:
    - name: backup
      image: busybox:1.36
      command: ["sleep", "600"]
      volumeMounts:
        - name: data
          mountPath: /data
          readOnly: true
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: prometheus-grafana
POD
kubectl -n monitoring wait --for=condition=Ready pod/grafana-backup --timeout=120s
kubectl -n monitoring exec grafana-backup -- tar czf - -C /data . > "$OUT/grafana-data.tar.gz"
kubectl -n monitoring delete pod grafana-backup --wait=false

# 5. keep the last 7 backups
ls -1dt "$DEST"/*/ | tail -n +8 | xargs -r rm -rf
echo "backup written to $OUT"
```

```
mkdir -p ~/bin && chmod +x ~/bin/backup-homelab.sh
```

Design points:

- **Runs as root** because it reads `/etc/kubernetes`; the `trap` hands the files back to the admin user even if a step fails, so a failed run never leaves a root-only directory.
- **`umask 077` and mode 700:** the etcd snapshot contains every Secret in clear text (Slack webhook, Grafana password, deploy keys). The backup is as sensitive as the cluster.
- **The snapshot is made inside the etcd pod** with `etcdctl`, then moved out through the pod's data directory, which is a host path.
- **Grafana is read through a throwaway pod** because the Grafana image has no `tar`.

## Step 2 — Run it once by hand

```
sudo ~/bin/backup-homelab.sh
ls -lh /var/backups/homelab/*/
```

Four files appear in the newest directory, and `Snapshot saved at ...` is printed. Check that the snapshot is readable:

```
etcdctl snapshot status /var/backups/homelab/<timestamp>/etcd-snapshot.db -w table
```

The table shows hash, revision, total keys and size.

## Step 3 — Copy to the worker

The control-plane user needs passwordless SSH to the worker (already in place from the Ansible setup). Check it, then do the first copy by hand:

```
ssh -o BatchMode=yes <USER>@<WORKER_IP> 'hostname'
ssh <USER>@<WORKER_IP> 'mkdir -p -m 700 ~/backups-from-master'
rsync -a --delete /var/backups/homelab/ <USER>@<WORKER_IP>:backups-from-master/
ssh <USER>@<WORKER_IP> 'ls -lh ~/backups-from-master/*/'
```

The same files appear on the worker. `--delete` makes it a mirror, so the worker keeps the same seven backups as the control plane.

## Step 4 — Schedule both

The backup runs as root (it needs `/etc/kubernetes`); the copy runs as the admin user, who owns the SSH key. Server time is UTC.

```
echo '15 3 * * * root /home/<USER>/bin/backup-homelab.sh >> /var/log/homelab-backup.log 2>&1' | sudo tee /etc/cron.d/homelab-backup
sudo chmod 644 /etc/cron.d/homelab-backup
(crontab -l 2>/dev/null; echo '30 3 * * * rsync -a --delete /var/backups/homelab/ <USER>@<WORKER_IP>:backups-from-master/ >> /home/<USER>/backup-rsync.log 2>&1') | crontab -
```

Run the cron command once by hand first (`sudo bash -c '~/bin/backup-homelab.sh >> /var/log/homelab-backup.log 2>&1'`), so the log file and its path are known to work before the first night.

## Step 5 — Helm values into Git

The values files of the Helm releases lived only in `~/k8s-manifests/`, a plain directory. They now live in `platform/` of the `homelab-gitops` repository:

```
platform/
  alloy/values.yaml
  argocd/values.yaml
  local-path-provisioner/values.yaml
  loki/values.yaml
  kube-prometheus-stack/values.yaml
  metallb/config.yaml
  ingress/argocd-ingress.yaml
  ingress/grafana-ingress.yaml
  README.md
```

`platform/README.md` lists each release with its chart version. Before copying, check that no file holds a secret:

```
grep -inE 'password|passwd|token|secret|api_url|webhook|key' ~/k8s-manifests/*values.yaml
```

Only references to Secrets by name appear (`existingSecret`, `api_url_file`); the Secrets themselves (`grafana-admin`, `alertmanager-slack`) are created by hand and never committed. `ingress-nginx` and `metallb` were installed with default values (`helm get values` returns `null`), so only the MetalLB address pool is stored.

ArgoCD still watches only `apps/my-site`, so these files change nothing in the cluster. Releases are upgraded by hand with `helm upgrade <release> <chart> --version <chart version> -f platform/<dir>/values.yaml`; the repository is the record of what is installed.

## Step 6 — A volume for Alertmanager

Alertmanager kept its silences and its record of sent notifications in the pod's temporary storage; a restart lost them. In `kube-prometheus-stack/values.yaml`, under `alertmanager.alertmanagerSpec` next to `secrets:`:

```
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
```

```
helm upgrade prometheus prometheus-community/kube-prometheus-stack -n monitoring --version 91.4.1 -f ~/k8s-manifests/prometheus-values.yaml
kubectl -n monitoring get pods -l app.kubernetes.io/name=alertmanager
kubectl -n monitoring get pvc | grep alertmanager
```

The pod is recreated and returns to `2/2 Running`; the claim is `Bound`, `1Gi`, `local-path`. The claim name is long because the operator builds it from the release name and the chart name (`alertmanager-<object name>-db-<pod name>`); it is not worth renaming the release to shorten it.

## Step 7 — Make control-plane metrics survive upgrades

[SRV-16 Step 8](./SRV-16-Logging-and-Alerting.md) made etcd, the scheduler, the controller manager and kube-proxy reachable by editing static pod manifests on the node. `kubeadm upgrade` regenerates those manifests, so the edits would be lost and four Prometheus targets would go down again. The same settings belong in the `kubeadm-config` ConfigMap, which kubeadm reads during an upgrade.

Back it up and prepare the new configuration in a file first:

```
kubectl -n kube-system get cm kubeadm-config -o yaml > ~/kubeadm-config.backup.yaml
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > cc.old.yaml
cp cc.old.yaml cc.new.yaml
```

In `cc.new.yaml` (API `kubeadm.k8s.io/v1beta4`, where `extraArgs` is a list of `name`/`value`), replace `controllerManager: {}` and `scheduler: {}` and add `extraArgs` under `etcd.local`:

```
controllerManager:
  extraArgs:
  - name: bind-address
    value: 0.0.0.0
scheduler:
  extraArgs:
  - name: bind-address
    value: 0.0.0.0
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
    - name: listen-metrics-urls
      value: http://0.0.0.0:2381
```

Compare, then write only the `ClusterConfiguration` key back:

```
diff cc.old.yaml cc.new.yaml
kubectl -n kube-system patch cm kubeadm-config --type merge -p "$(python3 -c 'import json;print(json.dumps({"data":{"ClusterConfiguration":open("cc.new.yaml").read()}}))')"
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep -B1 -A3 extraArgs
```

Three `extraArgs` blocks appear. Nothing restarts: the running cluster is unchanged, and the values apply at the next `kubeadm upgrade`.

**kube-proxy is not covered by this ConfigMap.** Its `metricsBindAddress` lives in the separate `kube-proxy` ConfigMap, which kubeadm could have reset during an upgrade. It was checked after the 1.32 and 1.33 upgrades and kept its value ([SRV-19](./SRV-19-Cluster-Upgrade.md)); it is still worth a look after every upgrade.

## Verification

After the first night (and any time later):

```
tail -3 /var/log/homelab-backup.log
ls -1 /var/backups/homelab/
cat ~/backup-rsync.log
```

and on the worker:

```
ls -1 ~/backups-from-master/
```

The log ends with `backup written to ...`, both machines list the same directories (at most seven), and the rsync log is empty. After seven nights both machines held seven identical directories of 336 MB in total, and the oldest manual backup had been rotated out.

## Limits

- **Restore tested only partly.** The etcd snapshot was restored into a scratch directory ([SRV-19 Step 1](./SRV-19-Cluster-Upgrade.md#step-1--test-a-restore-first)); a full control-plane rebuild has not been rehearsed.
- **Same site, same network.** Both copies are in the same room. A fire, a theft or a power surge takes both.
- **`rsync --delete` mirrors mistakes too.** If the source directory is emptied or deleted, the next night's copy empties the worker as well. The seven-day window protects against a bad day only if someone notices within a day.
- **Secrets in the clear.** The etcd snapshot and the `pki` archive can rebuild the cluster and read every Secret; the directory modes (700) are the only protection. Do not copy the backups to anything shared.
- **Not covered:** Prometheus and Loki data (regenerated), and the data of any later stateful application, which needs its own backup.
- **Cron time is server time (UTC).** The two jobs are 15 minutes apart; if the backup ever takes longer than that, the copy runs on a half-written directory.

## Open items

- 🟡 **Restore test.** The snapshot restores into a scratch directory ([SRV-19](./SRV-19-Cluster-Upgrade.md#step-1--test-a-restore-first)); a full control-plane rebuild is still to be rehearsed.
- ⬜ **Off-site copy** of the backups, encrypted, to cloud storage or another location.
- ⬜ **Alert when a backup fails or is stale.** Today a missed night is noticed only by looking at the log.
- ✅ **`metricsBindAddress` of kube-proxy** survived both cluster upgrades ([SRV-19](./SRV-19-Cluster-Upgrade.md)).
- ⬜ **Move the Helm releases under ArgoCD,** or keep upgrading them by hand from `platform/` (decision pending).

## Files

| File | Location | Contents |
|---|---|---|
| `backup-homelab.sh` | `~/bin/` on `<HOSTNAME>` | The backup script |
| `homelab-backup` | `/etc/cron.d/` on `<HOSTNAME>` | Root job, 03:15 |
| User crontab entry | `<HOSTNAME>` | `rsync` to the worker, 03:30 |
| `homelab-backup.log` | `/var/log/` on `<HOSTNAME>` | Output of the backup job |
| `backup-rsync.log` | `~` on `<HOSTNAME>` | Output of the copy job |
| `/var/backups/homelab/` | `<HOSTNAME>` | Seven newest backups |
| `~/backups-from-master/` | `<WORKER_HOSTNAME>` | Mirror of the above |
| `platform/` | `homelab-gitops` repository | Helm values and config |
| `kubeadm-config.backup.yaml` | `~` on `<HOSTNAME>` | Copy of the ConfigMap before Step 7 |

## Related

- [Dedicated GPU Node](./SRV-17-Dedicated-GPU-Node.md) — the previous step
- [Persistent Storage](./SRV-15-Persistent-Storage.md) — the volumes this protects; its open items are now closed
- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — Alertmanager and the control-plane metrics fix that Steps 6 and 7 complete
- [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md) — the repository that now also holds `platform/`
