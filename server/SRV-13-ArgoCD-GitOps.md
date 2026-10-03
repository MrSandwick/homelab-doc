---
tags: [homelab, project, kubernetes, argocd, gitops]
---

# ArgoCD and GitOps

> Status: 🟢 **Working.** ArgoCD installed and reachable at `http://<INGRESS_IP>/argocd`; the `my-site` workload is synced from the public `homelab-gitops` repository with automated sync, prune and self-heal verified. 🟡 Open: admin password rotation, registry-hosted image for `my-site`, laptop-node scheduling policy — see [Open items](#open-items).

Picks up after [First Real Workload](./SRV-10-First-Real-Workload.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Contents

1. [Components](#components)
2. [End state](#end-state)
3. [Step 1 — Install ArgoCD](#step-1--install-argocd)
4. [Step 2 — Pod placement](#step-2--pod-placement)
5. [Step 3 — Serve ArgoCD under /argocd](#step-3--serve-argocd-under-argocd)
6. [Step 4 — GitOps repository](#step-4--gitops-repository)
7. [Step 5 — Push access](#step-5--push-access)
8. [Step 6 — ArgoCD Application](#step-6--argocd-application)
9. [Verification](#verification)
10. [Open items](#open-items)
11. [Files](#files)
12. [Related](#related)

Problems hit during this work are recorded separately in [SRV-13-TRBL](../troubleshooting/SRV-13-TRBL-ArgoCD-GitOps.md).

## Components

| Component | Installed as | Namespace / location | Role |
|---|---|---|---|
| ArgoCD | Helm release `argocd` (chart `argo/argo-cd`) | `argocd` | GitOps controller: reconciles the cluster toward the state in git |
| `argocd-ingress` | Ingress | `argocd` | `/argocd` → `argocd-server:80` |
| `homelab-gitops` | Public GitHub repository | `github.com/MrSandwick/homelab-gitops` | Source of truth for workload manifests |
| `my-site` | ArgoCD `Application` | `argocd` | Syncs `apps/my-site` from the repository into namespace `default` |

Git is the source of truth for `my-site` from this point: changes are made by commit, and manual changes in the cluster are reverted. Concepts: [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-GitOps-and-ArgoCD.md).

## End state

```
Browser (LAN) → <INGRESS_IP> (MetalLB) → ingress-nginx → Grafana (path: /)
                                          ├→ my-site (path: /site, on the worker)
                                          └→ ArgoCD (path: /argocd)

homelab-gitops (GitHub, main) ── polled by ArgoCD ──→ Application my-site ──→ namespace default
```

## Step 1 — Install ArgoCD

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd
```

## Step 2 — Pod placement

ArgoCD runs on `<WORKER_HOSTNAME>`. The control-plane taint excludes only the control plane; the laptop node (`<GPU_HOSTNAME>`) is untainted and is kept out by cordoning it, after which the ArgoCD pods are recreated on the worker:

```
kubectl cordon <GPU_HOSTNAME>
kubectl delete pods --all -n argocd
kubectl get pods -n argocd -o wide
```

**Decision:** the laptop stays in the cluster, cordoned, running only its node-level DaemonSet pods (CNI, kube-proxy, node-exporter). It is a Wi-Fi, battery-powered machine that sleeps — unsuitable for persistent workloads.

The pods initially landed on the laptop — see [SRV-13-TRBL](../troubleshooting/SRV-13-TRBL-ArgoCD-GitOps.md#argocd-pods-scheduled-on-an-unexpected-node).

## Step 3 — Serve ArgoCD under /argocd

ArgoCD serves its own TLS and assumes it is hosted at `/`. `~/k8s-manifests/argocd-values.yaml`:

```yaml
configs:
  params:
    server.insecure: true
    server.rootpath: /argocd
    server.basehref: /argocd
```

```
helm upgrade argocd argo/argo-cd -n argocd -f ~/k8s-manifests/argocd-values.yaml
```

| Value | Effect |
|---|---|
| `server.insecure` | Disables ArgoCD's built-in TLS; the ingress fronts it over HTTP |
| `server.rootpath`, `server.basehref` | ArgoCD serves and generates asset URLs under the `/argocd` prefix |

Ingress — no `rewrite-target` annotation (unlike `my-site` in [SRV-10](./SRV-10-First-Real-Workload.md)), since ArgoCD handles the prefix itself:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /argocd
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

`/` remains claimed by Grafana; ingress-nginx selects the longest matching prefix, so `/argocd` routes to ArgoCD.

Verified: login page loads at `http://<INGRESS_IP>/argocd`. The initial admin password is read from the `argocd-initial-admin-secret` Secret in the `argocd` namespace.

## Step 4 — GitOps repository

Public repository `homelab-gitops`. The manifests contain no secrets; secrets must never be committed to it.

```
homelab-gitops/
└── apps/
    └── my-site/
        ├── my-site.yaml
        └── my-site-ingress.yaml
```

Git is initialized at the repository root, not inside `apps/`: the repository describes the whole cluster, leaving room for sibling directories (infrastructure values, ArgoCD `Application` manifests). The `Application`'s `path: apps/my-site` is relative to that root.

## Step 5 — Push access

Pushes to `homelab-gitops` go over HTTPS, authenticated with a fine-grained Personal Access Token scoped to the repository with **Contents: Read and write**. GitHub does not accept the account password for git operations.

**Alternative not used:** an SSH deploy key with write access — does not expire, better suited to a headless server.

Two push failures preceded the working setup — see [SRV-13-TRBL](../troubleshooting/SRV-13-TRBL-ArgoCD-GitOps.md#github-push-password-authentication-rejected).

## Step 6 — ArgoCD Application

Kept in `~/k8s-manifests/` and applied once by hand (not stored in the gitops repository):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-site
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/MrSandwick/homelab-gitops.git
    targetRevision: main
    path: apps/my-site
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

| Field | Meaning |
|---|---|
| `source` | Where the manifests live (repository, branch, path) |
| `destination` | Where they are applied (this cluster, namespace `default`) |
| `syncPolicy.automated` | ArgoCD applies git changes without a manual sync |
| `prune: true` | Resources removed from git are deleted from the cluster |
| `selfHeal: true` | Manual changes in the cluster are reverted to match git |

Two distinct sets of files: the manifests in the repository (what to run) and the `Application` object (tells ArgoCD to watch that repository). Without the `Application`, ArgoCD is installed but watches nothing. The repository is `homelab-gitops`; `my-site` is only the application's name within it.

Not implemented: storing the `Application` manifests in the repository as well (app-of-apps).

## Verification

| Check | Result |
|---|---|
| `http://<INGRESS_IP>/argocd` | Login page served |
| `kubectl get pods -n argocd -o wide` | All pods on `<WORKER_HOSTNAME>` |
| Application `my-site` | Synced / Healthy; adopted the already-running resources |
| `replicas` changed in git and pushed | Picked up by ArgoCD (polls roughly every 3 minutes; UI **Refresh** forces it) |
| `kubectl scale` on the Deployment by hand | Reverted to the git value (self-heal) |

## Open items

- ⬜ **Admin password:** change the ArgoCD admin password and delete `argocd-initial-admin-secret`. Not yet done.
- ⬜ **`my-site` image:** still imported only into the worker's containerd ([SRV-10](./SRV-10-First-Real-Workload.md)) — a weak point under GitOps. Move it to a registry (GHCR or Docker Hub), then drop `imagePullPolicy: Never` and the `nodeSelector`.
- ⬜ **Laptop node:** absent from the Ansible inventory ([SRV-09](./SRV-09-Ansible-Node-Provisioning.md)); decide whether it belongs there. The standing-cordon decision above also needs reconciling with [SRV-11](./SRV-11-GPU-Laptop-Node-Profile.md), which describes cordon as a temporary practice and expects GPU workloads on this node.

## Files

| File | Location | Contents |
|---|---|---|
| `argocd-values.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | Helm values: `insecure`, `rootpath`, `basehref` |
| ArgoCD `Application` manifest | `~/k8s-manifests/` on `<HOSTNAME>` | `Application` `my-site` |
| `apps/my-site/my-site.yaml`, `apps/my-site/my-site-ingress.yaml` | `homelab-gitops` repository | Deployment + Service, Ingress |

## Related

- [SRV-13-TRBL](../troubleshooting/SRV-13-TRBL-ArgoCD-GitOps.md) — troubleshooting for this doc
- [First Real Workload](./SRV-10-First-Real-Workload.md) — the workload now managed through git
- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — the ingress entry point ArgoCD is served through
- [Node Profile — GPU Laptop Worker](./SRV-11-GPU-Laptop-Node-Profile.md) — the third node the pods first landed on
- [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md) — inventory that does not yet include the laptop
- [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-GitOps-and-ArgoCD.md) — companion Guides repository
- [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md) — companion Guides repository
