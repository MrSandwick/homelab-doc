---
tags: [homelab, project, kubernetes, workload, ingress, scheduling]
---

# First Real Workload

> Status: 🟢 **Complete.** Static site (nginx) served through the existing ingress-nginx + MetalLB entry point at `/site`, scheduled on the worker node. Closes the last open item in [SRV-00](./SRV-00-Project-Overview.md).

Picks up after [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Contents

1. [Components](#components)
2. [End state](#end-state)
3. [Step 1 — Build the image](#step-1--build-the-image)
4. [Step 2 — Import the image into containerd](#step-2--import-the-image-into-containerd)
5. [Step 3 — First deployment: control plane (reverted)](#step-3--first-deployment-control-plane-reverted)
6. [Step 4 — Redeployment on the worker](#step-4--redeployment-on-the-worker)
7. [Step 5 — Route through the Ingress](#step-5--route-through-the-ingress)
8. [Verification](#verification)
9. [Files](#files)
10. [Related](#related)

## Components

No new cluster software; this entry adds Kubernetes objects on top of [SRV-08](./SRV-08-Helm-Observability-Ingress.md).

| Name | Kind | Namespace | Runs on | Purpose |
|---|---|---|---|---|
| `my-site:v1` | Container image (`nginx:alpine` + `index.html`) | — | worker's containerd | The site |
| `my-site` | Deployment (1 replica) | `default` | worker (pinned) | Runs the pod |
| `my-site` | Service (port 80) | `default` | — | Stable address for the pod |
| `my-site-ingress` | Ingress | `default` | — | `/site` → `my-site:80` |

Overview of each layer: [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md).

## End state

```
Browser (LAN) → <INGRESS_IP> (MetalLB) → ingress-nginx → Grafana (path: /)
                                          ├→ my-site (path: /site, on <WORKER_HOSTNAME>)
                                          ↳ Prometheus + Alertmanager + node-exporter (×2) feeding Grafana
```

## Step 1 — Build the image

`~/my-site/Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

```
cd ~/my-site
docker build -t my-site:v1 .
```

## Step 2 — Import the image into containerd

No registry is used; the Docker-built image was imported into containerd's `k8s.io` namespace directly. See [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Local-Container-Images-Without-a-Registry.md).

```
docker save my-site:v1 | sudo ctr -n k8s.io images import -
sudo ctr -n k8s.io images list | grep my-site
```

Listed with `io.cri-containerd.image=managed`. The image exists only on the node it was imported on, so the Deployment must be pinned to that node.

## Step 3 — First deployment: control plane (reverted)

### ⚠️ Real mistake made here

**What happened:** the image had been imported on the control-plane node, so the first Deployment was pinned there:

```yaml
nodeSelector:
  kubernetes.io/hostname: <HOSTNAME>
```

The pod stayed `Pending`:

```
0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }
```

A toleration was added and the pod ran:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

**Root cause:** node placement followed where the image had been imported, not where the workload belonged. The result put an application on the control plane, contrary to the taint restored in [Kubernetes Installation, Step 10](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint).

**Resolution:** everything deployed so far was removed and redone on the worker (Step 4):

```
kubectl delete -f deployment.yaml
kubectl delete -f ingress.yaml
```

## Step 4 — Redeployment on the worker

Site source copied to the worker, then the image rebuilt and imported **on the worker**:

```
scp -r ~/my-site <USERNAME>@<WORKER_IP>:~/my-site
```

On `<WORKER_HOSTNAME>`:

```
cd ~/my-site
docker build -t my-site:v1 .
docker save my-site:v1 | sudo ctr -n k8s.io images import -
```

`deployment.yaml` — `nodeSelector` pointed at the worker, `tolerations` removed:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-site
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-site
  template:
    metadata:
      labels:
        app: my-site
    spec:
      nodeSelector:
        kubernetes.io/hostname: <WORKER_HOSTNAME>
      containers:
        - name: my-site
          image: docker.io/library/my-site:v1
          imagePullPolicy: Never
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-site
spec:
  selector:
    app: my-site
  ports:
    - port: 80
      targetPort: 80
```

`imagePullPolicy: Never` forces use of the locally imported image; the image reference uses the fully-qualified name containerd assigns on import (`docker.io/library/my-site:v1`).

```
kubectl apply -f deployment.yaml
kubectl get pods -l app=my-site -o wide
```

`NODE`: `<WORKER_HOSTNAME>`.

## Step 5 — Route through the Ingress

Added as a second path on the existing ingress-nginx entry point ([SRV-08](./SRV-08-Helm-Observability-Ingress.md)). `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-site-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /site
            pathType: Prefix
            backend:
              service:
                name: my-site
                port:
                  number: 80
```

```
kubectl apply -f ingress.yaml
```

- Separate Ingress object in `default` (Grafana's is in `monitoring`); both are served by the same controller and address.
- `rewrite-target: /` rewrites all `/site*` requests to `/` — adequate for the current single page; sub-paths or assets would require a capture-group rewrite.

## Verification

| Check | Result |
|---|---|
| `kubectl get pods -l app=my-site -o wide` | Pod `Running` on `<WORKER_HOSTNAME>` |
| `http://<INGRESS_IP>/site` from a LAN browser | Site served |
| `http://<INGRESS_IP>/` from a LAN browser | Grafana still served |
| Control-plane taint | Unchanged; no tolerations on `my-site` |

## Files

| File | Node | Contents |
|---|---|---|
| `~/my-site/index.html`, `~/my-site/Dockerfile` | both (source copied to the worker) | Site and image build |
| `deployment.yaml` | control plane | Deployment + Service |
| `ingress.yaml` | control plane | Ingress `/site` → `my-site:80` |

## Related

- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — the ingress and MetalLB entry point this reuses
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the control-plane taint, and why it was restored
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the worker this runs on
- [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md) — companion Guides repository
- [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Local-Container-Images-Without-a-Registry.md) — companion Guides repository
- [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
- [Guide: What Is a Cluster](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Homelab-Clusters.md) — companion Guides repository — workload placement and choice of first workload
