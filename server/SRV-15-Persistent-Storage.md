---
tags: [homelab, project, kubernetes, storage, helm, grafana, prometheus]
---

# Persistent Storage

> Status: 🟢 **Working.** `local-path-provisioner` gives the cluster a default StorageClass; Grafana and Prometheus now keep their data on persistent volumes, and Grafana's admin password survived a pod restart. 🟡 Open: the volumes are not backed up, and Alertmanager still has no volume — see [Open items](#open-items).

Picks up after [CI/CD with GitHub Actions and GHCR](./SRV-14-CI-CD-GHCR.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Step 1 — Install local-path-provisioner](#step-1--install-local-path-provisioner)
4. [Step 2 — Grafana: admin secret and volume](#step-2--grafana-admin-secret-and-volume)
5. [Step 3 — Prometheus volume](#step-3--prometheus-volume)
6. [Verification](#verification)
7. [Limits of this storage](#limits-of-this-storage)
8. [Open items](#open-items)
9. [Files](#files)
10. [Related](#related)

## Components

| Component | Installed as | Namespace / location | Role |
|---|---|---|---|
| `local-path-provisioner` | Helm release `local-path-provisioner` (chart from the project's repository, tag `v0.0.37`) | `local-path-storage` | Creates a volume as a directory on a node's disk when a pod asks for one |
| StorageClass `local-path` | Created by the chart, marked default | Cluster-wide | The "vending machine": a claim without an explicit class is served by it |
| PVC `prometheus-grafana` | Created by the Grafana chart | `monitoring` | 5 GiB for Grafana's database |
| PVC for Prometheus | Created from `volumeClaimTemplate` | `monitoring` | 10 GiB for Prometheus's metrics |
| Secret `grafana-admin` | Created by hand | `monitoring` | Grafana admin user and password |

Concepts (volume, claim, class): [Guide: Persistent Storage in Kubernetes](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Persistent-Storage-PV-PVC-StorageClass.md).

## Why

A pod has no persistent disk by default: everything it writes lives inside the container and is lost when the pod restarts, is recreated or moves. Before this step:

```
kubectl get pvc -n monitoring     →  No resources found in monitoring namespace.
kubectl get storageclass          →  No resources found
```

Consequences for what was already running:

- **Grafana** keeps users, dashboards and the admin password in its own database. The admin password is written into that database at first start; the value from the Secret is applied only then. Without a volume the database is recreated at every pod restart, so a password changed in the UI silently reverts to the Secret's value.
- **Prometheus** loses its whole metric history at every restart.

## Step 1 — Install local-path-provisioner

The project ships a Helm chart in its own repository (`deploy/chart/`); a hosted Helm repository for it was not found, so the chart is installed from a clone at the release tag. Helm is used so that it matches the other components ([SRV-08](./SRV-08-Helm-Observability-Ingress.md), [SRV-13](./SRV-13-ArgoCD-GitOps.md)).

```
cd ~
git clone --depth 1 --branch v0.0.37 https://github.com/rancher/local-path-provisioner.git
nano ~/k8s-manifests/local-path-values.yaml
```

```yaml
storageClass:
  defaultClass: true
```

```
helm install local-path-provisioner ~/local-path-provisioner/deploy/chart/local-path-provisioner \
  --namespace local-path-storage --create-namespace \
  -f ~/k8s-manifests/local-path-values.yaml
```

`defaultClass: true` marks `local-path` as the default StorageClass, so a PVC with no class is served by it. The clone is kept: `helm upgrade` needs the chart directory.

Result:

```
helm list -n local-path-storage            local-path-provisioner  ...  deployed  local-path-provisioner-0.0.37  v0.0.37
kubectl -n local-path-storage get pods     1/1 Running on <WORKER_HOSTNAME>
kubectl get storageclass                   local-path (default)  ...  Delete  WaitForFirstConsumer
```

## Step 2 — Grafana: admin secret and volume

The admin password has to be in place **before** the first start with a volume, because that start writes it into the new database. The Secret is created by hand and the password is read without echo:

```
read -s -p "Grafana admin password: " GPW; echo
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$GPW"
unset GPW
```

`~/k8s-manifests/prometheus-values.yaml`:

```yaml
grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
  persistence:
    enabled: true
    type: pvc
    storageClassName: local-path
    accessModes: [ReadWriteOnce]
    size: 5Gi
```

The release had no user-supplied values (`helm get values prometheus -n monitoring` printed `null`), so this file is now the complete record of overrides. The upgrade is pinned to the installed chart version (`kube-prometheus-stack-91.4.1`), so Helm does not pick up a newer chart as a side effect. A dry run goes first:

```
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --version 91.4.1 -f ~/k8s-manifests/prometheus-values.yaml \
  --dry-run > /dev/null && echo "dry-run OK"

helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --version 91.4.1 -f ~/k8s-manifests/prometheus-values.yaml
```

Grafana restarts; anything created by hand in the UI before this point is lost, because it was never stored. The password hints at the end of the `helm upgrade` output (`kubectl get secrets prometheus-grafana …`) are stale: they refer to the chart's own generated Secret, no longer used.

## Step 3 — Prometheus volume

Appended to the same values file, at the top level next to `grafana:`:

```yaml
prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: 8GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

| Value | Effect |
|---|---|
| `retention: 15d` | Metrics are kept for 15 days |
| `retentionSize: 8GB` | The database is capped below the 10 GiB volume, so the volume cannot fill up |
| `volumeClaimTemplate` | The operator creates the claim for the Prometheus StatefulSet |

The same `helm upgrade` command (now revision 3) applies it. Prometheus restarts and its previous history is gone; it was never stored.

## Verification

| Check | Result |
|---|---|
| `helm list -n local-path-storage` | Release `deployed`, chart `0.0.37` |
| `kubectl get storageclass` | `local-path (default)`, `WaitForFirstConsumer`, reclaim `Delete` |
| `kubectl -n monitoring get pvc` | `prometheus-grafana` `Bound`, 5Gi, `local-path`; the Prometheus claim `Bound`, 10Gi, `local-path` |
| Grafana pod | `3/3 Running` on `<WORKER_HOSTNAME>` |
| `kubectl -n monitoring rollout restart deployment prometheus-grafana`, then login with the new password | Password kept after the restart |

## Limits of this storage

- **The data is a directory on one node's disk** (`/opt/local-path-provisioner` on `<WORKER_HOSTNAME>`). A pod with such a volume can only run on that node; for Grafana and Prometheus that is where everything already runs.
- **It is not a backup or a replica.** If the disk is lost, the data is lost.
- **`WaitForFirstConsumer`:** a volume is created only when a pod that uses the claim is scheduled, so a new claim shows `Pending` until then. This is normal.
- **Reclaim policy `Delete`:** deleting a claim deletes its data.

**Alternatives not used:** hand-written PersistentVolumes (the provisioner does that on demand); NFS (needs a separate storage host — see [Guide: NAS vs Server](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/decisions/Guide-NAS-vs-Server.md)); a replicated block store such as Longhorn (replication needs more than one schedulable node with spare capacity, and only one is available).

## Open items

- ⬜ **No backup** of the volumes. The data is easy to regenerate (metrics) or small (Grafana state), but nothing protects it.
- ⬜ **Alertmanager** has no volume; its silences and notification log are lost on restart. Alert routing now exists ([SRV-16](./SRV-16-Logging-and-Alerting.md)), so this is a real limit.
- ⬜ **Helm releases are applied by hand,** with their values files in `~/k8s-manifests/`, not through ArgoCD. Moving them into `homelab-gitops` would make the whole platform GitOps-managed.

## Files

| File | Location | Contents |
|---|---|---|
| `local-path-values.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | `storageClass.defaultClass: true` |
| `prometheus-values.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | Grafana admin Secret reference and volume, Prometheus volume and retention; later also the Loki data source and Alertmanager routing ([SRV-16](./SRV-16-Logging-and-Alerting.md)) |
| `~/local-path-provisioner/` | `<HOSTNAME>` | Clone at tag `v0.0.37`; the chart directory for `helm upgrade` |
| Secret `grafana-admin` | Namespace `monitoring` | Created by hand; never committed |

## Related

- [CI/CD with GitHub Actions and GHCR](./SRV-14-CI-CD-GHCR.md) — the previous step
- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — the next step; Loki also uses a `local-path` volume
- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — where `kube-prometheus-stack` and Grafana were first installed
- [Guide: Persistent Storage in Kubernetes](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Persistent-Storage-PV-PVC-StorageClass.md) — companion Guides repository
- [Guide: Helm Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Helm-Basics.md) — companion Guides repository
