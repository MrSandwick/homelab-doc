---
tags: [homelab, project, kubernetes, helm, observability, ingress]
---

# Helm, Observability, and Ingress

> Status: 🟢 **Working.** Helm installed; `kube-prometheus-stack` collecting metrics from both nodes; Grafana reachable from the LAN through ingress-nginx at a MetalLB-assigned address (`<INGRESS_IP>`).

Picks up after [Cluster Verification](./SRV-07-Cluster-Verification.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Contents

1. [Components installed](#components-installed)
2. [End state](#end-state)
3. [Step 1 — Helm](#step-1--helm)
4. [Step 2 — Monitoring: kube-prometheus-stack](#step-2--monitoring-kube-prometheus-stack)
5. [Step 3 — Ingress controller: ingress-nginx](#step-3--ingress-controller-ingress-nginx)
6. [Step 4 — Load balancer: MetalLB](#step-4--load-balancer-metallb)
7. [Step 5 — Grafana through the Ingress](#step-5--grafana-through-the-ingress)
8. [Files created](#files-created)
9. [Related](#related)

Problems hit during this work are recorded separately in [SRV-08-TRBL](../troubleshooting/SRV-08-TRBL-Helm-Observability-Ingress.md).

## Components installed

| Component | Installed as | Namespace | Role |
|---|---|---|---|
| Helm 3 | Binary on the control plane | — | Installs and tracks the three charts below |
| kube-prometheus-stack | Helm release `prometheus` | `monitoring` | Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics |
| ingress-nginx | Helm release `ingress-nginx` | `ingress-nginx` | HTTP routing by path/host to Services |
| MetalLB | Helm release `metallb` | `metallb-system` | LAN IP for `LoadBalancer` Services (L2 mode) |

Overview of each component: [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md).

## End state

```
Browser (LAN) → <INGRESS_IP> (MetalLB) → ingress-nginx → Grafana (path: /)
                                          ↳ Prometheus + Alertmanager + node-exporter (×2) feeding it
```

`/` is claimed by Grafana; further services require distinct paths or hostnames.

## Step 1 — Helm

### Install

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

The script needs working DNS for `raw.githubusercontent.com`; the first run failed on the ISP DNS blocking — see [SRV-08-TRBL](../troubleshooting/SRV-08-TRBL-Helm-Observability-Ingress.md#helm-install-script-failed-to-resolve).

### Chart repositories

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Smoke test

Disposable release before installing anything real:

```
helm install nginx-test bitnami/nginx
kubectl get pods
kubectl get svc
helm uninstall nginx-test
```

Pod and Service came up; `helm uninstall` removed both.

## Step 2 — Monitoring: kube-prometheus-stack

### Install

```
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### node-exporter on both nodes

node-exporter (DaemonSet) came up on both nodes with the control-plane `NoSchedule` taint left in place — the chart's DaemonSet carries the toleration. See [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md).

### Initial access

Before the ingress existed, Grafana was reached via port-forward (`--address 0.0.0.0` required for LAN access, as in [Kubernetes Installation, Step 10](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint)):

```
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address 0.0.0.0
```

Admin password (chart-generated Secret):

```
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

## Step 3 — Ingress controller: ingress-nginx

### Install

```
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

The install must be left to complete before running further commands in the same session; an interrupted first attempt left a `failed` release behind — see [SRV-08-TRBL](../troubleshooting/SRV-08-TRBL-Helm-Observability-Ingress.md#helm-install-interrupted-cannot-re-use-a-name).

## Step 4 — Load balancer: MetalLB

Required for the ingress controller's `LoadBalancer` Service to receive an external IP on bare metal. See [Guide: Ingress and MetalLB](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ingress-and-MetalLB.md).

### Install

```
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb \
  --namespace metallb-system --create-namespace
```

### Address pool

Pool: `<METALLB_POOL_START>`–`<METALLB_POOL_END>` (`<ADMIN_SUBNET>.240`–`.250`), outside the Admin VLAN DHCP range `.100`–`.200` ([Router Configuration](../network/NET-04-Router-Configuration.md)). Router admin access was unavailable to confirm the DHCP settings directly, so the range was verified with a ping sweep:

```
for i in {240..250}; do
  ip=<ADMIN_SUBNET>.$i
  ping -c 1 -W 1 $ip | grep "bytes from" && echo "$ip IN USE"
done
```

No replies — range free.

### Pool configuration

Manifests are kept in `~/k8s-manifests/` on the control-plane node (convention adopted from this point). `metallb-config.yaml` — L2 mode:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - <METALLB_POOL_START>-<METALLB_POOL_END>
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

```
kubectl apply -f ~/k8s-manifests/metallb-config.yaml
```

### Verification

```
kubectl get svc -n ingress-nginx
```

`ingress-nginx-controller` `EXTERNAL-IP`: `<pending>` → `<INGRESS_IP>` (first address in the pool) immediately after apply.

## Step 5 — Grafana through the Ingress

`~/k8s-manifests/grafana-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
```

```
kubectl apply -f ~/k8s-manifests/grafana-ingress.yaml
```

Verified from another LAN machine: `http://<INGRESS_IP>` serves Grafana without port-forward. Dashboards **Node Exporter / Nodes** and **Kubernetes / Compute Resources / Cluster** show live data for both nodes.

## Files created

| File (on `<HOSTNAME>`) | Contents |
|---|---|
| `~/k8s-manifests/metallb-config.yaml` | `IPAddressPool` + `L2Advertisement` |
| `~/k8s-manifests/grafana-ingress.yaml` | Ingress: `/` → `prometheus-grafana:80` |

## Related

- [SRV-08-TRBL](../troubleshooting/SRV-08-TRBL-Helm-Observability-Ingress.md) — troubleshooting for this doc
- [Cluster Verification](./SRV-07-Cluster-Verification.md) — the state of the cluster this builds on
- [Network Configuration](./SRV-03-Network-Configuration.md) — the router-DNS configuration Helm's install script depends on
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the original `port-forward` usage
- [Router Configuration](../network/NET-04-Router-Configuration.md) — the Admin VLAN DHCP pool the MetalLB range sits outside of
- [First Real Workload](./SRV-10-First-Real-Workload.md) — the second route added to this ingress
- [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md) — companion Guides repository
- [Guide: Helm Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Helm-Basics.md) — companion Guides repository
- [Guide: Ingress and MetalLB](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ingress-and-MetalLB.md) — companion Guides repository
- [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
