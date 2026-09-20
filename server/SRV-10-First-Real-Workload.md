---
tags: [homelab, project, kubernetes, workload, ingress, scheduling]
---

# First Real Workload

> Status: 🟢 **Complete — first real application workload deployed and verified, closing the last open item from [SRV-00](./SRV-00-Project-Overview.md).** A static site is served through the existing ingress-nginx + MetalLB entry point on a new `/site` path, running on the worker node while the control plane stays free of application load.

Picks up after [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Why a static site first

A static site is the simplest workload that still exercises the whole path: build an image, get it onto a node, schedule a pod, put a Service in front of it, and route to it from the LAN. It is **stateless** — no database, no persistent volume — so if something fails, the cause is in the path being tested rather than in storage. Anything with persistent state (Nextcloud, a Minecraft world) comes after this path is proven.

## Building the image

A minimal `index.html` and a `Dockerfile` in `~/my-site`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

```
cd ~/my-site
docker build -t my-site:v1 .
```

## Getting a local image into containerd (no registry)

The image was built locally with Docker and not pushed to any registry (Docker Hub, GHCR…). Docker and the containerd instance Kubernetes uses keep **separate image stores**, so Kubernetes cannot see a Docker-built image on its own (background: [Docker vs Containerd](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Docker-vs-Containerd.md)). It was imported into containerd's `k8s.io` namespace directly:

```
docker save my-site:v1 | sudo ctr -n k8s.io images import -
```

Verified:

```
sudo ctr -n k8s.io images list | grep my-site
```

The entry appeared with the label `io.cri-containerd.image=managed`, confirming Kubernetes' runtime can use it. See [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Local-Container-Images-Without-a-Registry.md).

**Consequence worth stating plainly:** an image imported this way exists **only in containerd on the one machine it was imported on.** A pod using it can be scheduled only onto that node — not the other — until the image is either pushed to a registry both nodes can pull from, or built and imported separately on every node that might run it.

### ⚠️ Real mistake made here: deployed to the control plane first

**What happened:** the image had been built and imported on the control-plane node, so the first Deployment pinned itself there:

```yaml
nodeSelector:
  kubernetes.io/hostname: <HOSTNAME>
```

That alone left the pod `Pending`, because the control plane carries the `node-role.kubernetes.io/control-plane:NoSchedule` taint. The scheduler's event said exactly that:

```
0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }
```

(The worker fails the `nodeSelector`; the control plane fails the taint.) A matching toleration got it running:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

**Why it was wrong anyway:** it worked, but it wasn't the intended end state. The control-plane taint was deliberately restored after the earlier nginx smoke test ([Kubernetes Installation](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint)) so that node stays reserved for cluster management, and a toleration was quietly undoing that for an ordinary application. Getting the pod to run had satisfied the wrong goal.

**Root cause:** the node was chosen because that's where the image happened to be — a constraint of the local-import approach — rather than because it was where the workload *should* run. The `Pending` event pointed at the taint, and the toleration was the fastest way past it, but the taint was telling the truth about where this workload belonged.

**Fix — reverted and redone on the worker:**

1. Removed what was deployed:
   ```
   kubectl delete -f deployment.yaml
   kubectl delete -f ingress.yaml
   ```
   (the Deployment and Service manifest, and the Ingress).
2. Copied the site source to the worker:
   ```
   scp -r ~/my-site <USERNAME>@<WORKER_IP>:~/my-site
   ```
3. Rebuilt and imported the image on the **worker** itself — the image doesn't transfer between nodes on its own:
   ```
   docker build -t my-site:v1 .
   docker save my-site:v1 | sudo ctr -n k8s.io images import -
   ```
4. Rewrote the manifest with `nodeSelector: kubernetes.io/hostname: <WORKER_HOSTNAME>` and **removed the `tolerations` block entirely** — the worker carries no taint, so none is needed.
5. Reapplied and checked where it landed:
   ```
   kubectl get pods -l app=my-site -o wide
   ```
   The `NODE` column showed the worker.

**Lesson:** a workload landing on the right node is not the same as it landing on *a* node. When a taint blocks a pod, ask whether the taint is wrong or whether the *placement* is. Here it was the placement.

## The final manifests (worker-targeted)

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

- **`nodeSelector`** pins the pod to the worker — the only node that has the image.
- **`imagePullPolicy: Never`** tells the kubelet to use the locally imported image and never try to pull it. It is the natural pairing with the local-import approach and makes the intent explicit. (With a versioned tag like `:v1` the default policy, `IfNotPresent`, would also have used the local copy; the trap is `:latest` or an untagged image, where the default is `Always` and the kubelet tries — and fails — to pull from a registry that doesn't hold it. `Never` also fails fast with `ErrImageNeverPull` on a node missing the image, instead of attempting a doomed pull.)
- **`docker.io/library/my-site:v1`** is the fully-qualified name containerd gives a bare `my-site:v1`; using it makes the reference match what was imported exactly.

## Exposing it through the existing Ingress

No new IP and no new ingress controller: the site was added as a second path on the entry point that already serves Grafana ([Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md)).

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

Two things worth knowing:

- This is a **second Ingress object** handled by the same controller, in the `default` namespace next to the Service it points at. An Ingress can only reference Services in its own namespace, which is why Grafana's rule (in `monitoring`) and this one (in `default`) are separate objects sharing one controller and one address.
- **`rewrite-target: /`** strips the `/site` prefix so the backend sees `/`, which is what a static `index.html` at the web root needs. The catch: it rewrites *every* matching path to `/`, so a site with multiple pages or assets under sub-paths would need a capture-group rewrite instead. Fine for a single page.

Verified end to end: `http://<INGRESS_IP>/site` from a browser on the LAN serves the page, and `/` still reaches Grafana — the first time this cluster has routed more than one backend through the ingress.

## Current traffic path

```
Browser (LAN) → <INGRESS_IP> (MetalLB) → ingress-nginx → Grafana (path: /)
                                          ├→ my-site (path: /site, on <WORKER_HOSTNAME>)
                                          ↳ Prometheus + Alertmanager + node-exporter (×2) feeding Grafana
```

## Related

- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — the ingress and MetalLB entry point this reuses
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the control-plane taint, and why it was restored
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the worker this runs on
- [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Local-Container-Images-Without-a-Registry.md) — companion Guides repository
- [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository — the taint/toleration mechanics behind the mistake above
