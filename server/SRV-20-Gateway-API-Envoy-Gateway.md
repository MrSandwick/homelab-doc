---
tags: [homelab, project, kubernetes, gateway-api, envoy, ingress, metallb, gitops]
---

# Gateway API with Envoy Gateway

> Status: 🟢 **Working.** `ingress-nginx`, which was retired upstream in March 2026, is replaced by Envoy Gateway and the Kubernetes Gateway API. The site, ArgoCD and Grafana answer on the same LAN address as before (`<INGRESS_IP>`), with the same paths; the routing is described by `Gateway` and `HTTPRoute` objects stored in `homelab-gitops`. 🟡 Open: plain HTTP only, one proxy replica — see [Open items](#open-items).

Picks up after [Cluster Upgrade 1.31 → 1.33](./SRV-19-Cluster-Upgrade.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Inventory the current routes](#step-1--inventory-the-current-routes)
5. [Step 2 — CRDs and the controller](#step-2--crds-and-the-controller)
6. [Step 3 — Gateway and routes](#step-3--gateway-and-routes)
7. [Step 4 — Compare old and new](#step-4--compare-old-and-new)
8. [Step 5 — Switch the address](#step-5--switch-the-address)
9. [Step 6 — Merge into Git](#step-6--merge-into-git)
10. [Verification](#verification)
11. [Limits](#limits)
12. [Open items](#open-items)
13. [Files](#files)
14. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Envoy Gateway v1.9.2 (release `eg`) | Namespace `envoy-gateway-system` | The controller: watches Gateway API objects and configures Envoy |
| Gateway API CRDs (standard channel) | Cluster | The object types `GatewayClass`, `Gateway`, `HTTPRoute`, … |
| `GatewayClass envoy` | Cluster | Binds the class name to the Envoy Gateway controller |
| `Gateway homelab` | `envoy-gateway-system` | The entry point: listener on port 80, LoadBalancer address from MetalLB |
| Envoy proxy pod | `envoy-gateway-system` | The data plane that actually carries the requests; created by the controller for the Gateway |
| `HTTPRoute argocd`, `grafana`, `my-site` | Namespaces of each application | Routing rules, owned by the application's namespace |

MetalLB is unchanged: it still answers "which LAN address reaches the cluster"; only the proxy behind that address is replaced.

## Why

`ingress-nginx` was the community Ingress controller maintained by Kubernetes SIG Network; it was retired in March 2026 and receives no more fixes, including security fixes. The `Ingress` API it implements is not removed, but it is frozen and every controller extended it with its own annotations. The Gateway API is its upstream successor.

Gateway API separates the roles. The platform side owns the `Gateway` (address, ports, which namespaces may attach); each application owns an `HTTPRoute` in its own namespace. Path rewriting, header matching and traffic splitting are part of the standard objects, not controller-specific annotations.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Implementation | Envoy Gateway | CNCF reference implementation built on Envoy, the proxy behind Istio and most cloud gateways; richest observability out of the box |
| Not chosen | NGINX Gateway Fabric | Good and lighter on memory, but less transferable than the Envoy ecosystem; the difference in memory does not matter at this size |
| Not chosen | Cilium Gateway API | Needs Cilium as the CNI; the cluster uses Flannel. Worth revisiting together with NetworkPolicy |
| Version | 1.9.2 | Supports Kubernetes 1.33–1.36, which is why the cluster was upgraded first ([SRV-19](./SRV-19-Cluster-Upgrade.md)) |
| CRDs | Installed separately, standard channel | The chart installs the experimental channel by default; applying the CRDs from the standard channel with `--server-side` keeps unstable resources out and makes CRD upgrades an explicit step |
| Rollout | Parallel on a second address, then switch | The old entry point kept working the whole time; the switch took about a minute |
| Gateway namespace | `envoy-gateway-system` | Next to the controller and its proxy pods |
| `allowedRoutes: All` | Any namespace may attach a route | Applications bring their own routes; the cluster is single-tenant |
| Route location | `argocd` and `grafana` routes in `platform/gateway-routes/`, `my-site` route in `apps/my-site/` | The site's route is part of the application that ArgoCD already syncs |

## Step 1 — Inventory the current routes

```
kubectl get ingress -A -o yaml
kubectl -n metallb-system get ipaddresspool,l2advertisement -o yaml
kubectl get svc -A | grep LoadBalancer
```

Findings:

- `argocd-ingress`: `/argocd` → `argocd-server:80`, no rewrite (ArgoCD serves itself under that prefix).
- `my-site-ingress`: `/site` → `my-site:80` with `rewrite-target: /`.
- `grafana-ingress`: `/` → `prometheus-grafana:80`, the catch-all.
- MetalLB pool with eleven addresses, only the first used by `ingress-nginx-controller`; any other address in the pool can serve as the test address.

## Step 2 — CRDs and the controller

```
helm template eg oci://docker.io/envoyproxy/gateway-crds-helm --version v1.9.2 \
  --set crds.gatewayAPI.enabled=true --set crds.gatewayAPI.channel=standard \
  --set crds.envoyGateway.enabled=true | kubectl apply --server-side -f -

helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.9.2 \
  -n envoy-gateway-system --create-namespace --set crds.enabled=false
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
kubectl get pods -n envoy-gateway-system
```

Twenty objects are created (Gateway API and Envoy Gateway CRDs plus a safe-upgrades admission policy). The controller pod is `Running`; the `certgen` job that generates its certificates shows `Completed`. Nothing changes for current traffic.

## Step 3 — Gateway and routes

`platform/envoy-gateway/gatewayclass.yaml` and `gateway.yaml`:

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: homelab
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

The controller creates an Envoy proxy and a `LoadBalancer` Service; MetalLB assigns the next free address. Routes live in the namespaces of their applications. `my-site` needs the equivalent of the old `rewrite-target: /`:

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-site
  namespace: default
spec:
  parentRefs:
    - name: homelab
      namespace: envoy-gateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /site
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: my-site
          port: 80
```

`ReplacePrefixMatch: /` strips the `/site` prefix and keeps the rest (`/site/x` reaches the backend as `/x`). The old annotation without a capture group sent every sub-path to `/`; this form is the precise behaviour. The `argocd` route matches `/argocd` with no filter, the `grafana` route matches `/`. A more specific prefix always wins over `/`, so the order of the rules does not matter.

```
kubectl apply -f platform/envoy-gateway/
kubectl wait --timeout=3m -n envoy-gateway-system gateway/homelab --for=condition=Programmed
kubectl apply -f platform/gateway-routes/ -f apps/my-site/my-site-httproute.yaml
kubectl get gateway -n envoy-gateway-system
kubectl get httproute -A
```

The Gateway reports `PROGRAMMED True` with a new address from the pool, and all three routes are listed.

## Step 4 — Compare old and new

Both entry points answer at the same time, so the paths can be compared one to one:

```
for ip in <INGRESS_IP> <NEW_IP>; do
  curl -s -o /dev/null -w "site:    %{http_code}\n" http://$ip/site/
  curl -s -o /dev/null -w "argocd:  %{http_code}\n" http://$ip/argocd/
  curl -s -o /dev/null -w "grafana: %{http_code}\n" http://$ip/
done
```

Identical codes on both: site `200`, ArgoCD `200`, Grafana `302` (redirect to the login page).

## Step 5 — Switch the address

The old address is the one in bookmarks, so the Gateway takes it over. Add an annotation to the Gateway that MetalLB reads and that Envoy Gateway copies onto the Service it creates:

```
spec:
  infrastructure:
    annotations:
      metallb.universe.tf/loadBalancerIPs: <INGRESS_IP>
```

The address is still held by `ingress-nginx`, so that goes first. The order matters: the old Ingress objects, then the controller, then the pin.

```
kubectl delete ingress argocd-ingress -n argocd
kubectl delete ingress grafana-ingress -n monitoring
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
kubectl apply -f platform/envoy-gateway/gateway.yaml
kubectl get gateway homelab -n envoy-gateway-system
kubectl get svc -n envoy-gateway-system | grep LoadBalancer
```

The Gateway and the Envoy Service report the old address. The outage on that address lasted about a minute; the test address answered throughout. The three checks from Step 4 now return the same codes on the old address.

`my-site-ingress` is managed by ArgoCD and is therefore not deleted by hand: deleting it would only make self-heal recreate it from Git. It is removed from the repository instead (next step).

## Step 6 — Merge into Git

The branch contains the Gateway files, the three routes and the removal of the three Ingress manifests (`apps/my-site/my-site-ingress.yaml`, `platform/ingress/*`), and `platform/README.md` lists the release. The merge is done after the switch so that ArgoCD does not delete the site's Ingress before the address has moved. After the merge ArgoCD removes the old Ingress and adopts the `HTTPRoute` that already exists in the cluster.

```
git checkout main && git pull
kubectl -n argocd get applications
kubectl get httproute -A
kubectl get ingress -A
```

`my-site` is `Synced` and `Healthy`, the three routes exist and no Ingress objects are left.

## Verification

```
kubectl get gateway -n envoy-gateway-system
kubectl get httproute -A
curl -s -o /dev/null -w "site: %{http_code}\n" http://<INGRESS_IP>/site/
```

The Gateway is `PROGRAMMED True` on the old address, three routes are accepted, the site answers `200`.

## Limits

- **Plain HTTP only.** There is no TLS listener; traffic on the LAN is unencrypted, as it was with `ingress-nginx`.
- **One proxy replica on one node.** Everything runs on the single worker ([SRV-17](./SRV-17-Dedicated-GPU-Node.md#node-roles)); if the worker is down, so is the entry point. The same limit as before the change.
- **The address pin depends on MetalLB's annotation.** If MetalLB's annotation name changes in a future release, the Service would get a different address from the pool; the Gateway's reported address shows it immediately.
- **Not covered:** rate limiting, authentication and other Envoy Gateway policies (`BackendTrafficPolicy`, `SecurityPolicy`) — available, not used.

## Open items

- ⬜ **TLS:** a certificate (for example from cert-manager) and an HTTPS listener on the Gateway.
- ⬜ **Envoy metrics and logs in Grafana:** access logs through Alloy/Loki and the proxy's metrics through a `ServiceMonitor`.
- ⬜ **More than one proxy replica,** once there is a second schedulable node.
- ✅ **Leftovers removed:** the old `*-ingress.yaml` files in `~/k8s-manifests/` on the control plane and the unused Helm repository `ingress-nginx`.
- ⬜ **Cilium** as CNI (Gateway API support, NetworkPolicy) — a separate project.

## Files

| File | Location | Contents |
|---|---|---|
| `gatewayclass.yaml`, `gateway.yaml` | `platform/envoy-gateway/` in `homelab-gitops` | GatewayClass and the `homelab` Gateway with the address pin |
| `argocd.yaml`, `grafana.yaml` | `platform/gateway-routes/` | Routes for ArgoCD and Grafana |
| `my-site-httproute.yaml` | `apps/my-site/` | Route for the site, with the prefix rewrite |
| `README.md` | `platform/` | Release `eg` with its chart version and how its CRDs were installed |
| Helm release `eg` | Namespace `envoy-gateway-system` | Installed by hand from `oci://docker.io/envoyproxy/gateway-helm` v1.9.2 with `crds.enabled=false` |

## Related

- [Cluster Upgrade 1.31 → 1.33](./SRV-19-Cluster-Upgrade.md) — why the cluster had to be upgraded first
- [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md) — `platform/` and the backup that covered the switch
- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — where `ingress-nginx` and MetalLB were first installed
- [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md) — what syncs `apps/my-site`
- [Guide: Gateway API and Envoy Gateway](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Gateway-API-and-Envoy-Gateway.md) — companion Guides repository
