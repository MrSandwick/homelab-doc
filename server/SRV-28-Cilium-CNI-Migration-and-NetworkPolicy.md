---
tags: [homelab, project, kubernetes, cilium, cni, networkpolicy, hubble, flannel, security]
---

# Cilium CNI Migration and NetworkPolicy

> Status: 🟢 **Working.** The pod network moved from Flannel to Cilium 1.20.4 on a live three-node cluster, node by node, with no change to the applications. `NetworkPolicy` is now enforced: a default-deny test blocks traffic, the nine policies that already existed (ArgoCD, Strimzi) are active, and the public portfolio namespace allows only `gateway → web → api → mongo`. Hubble shows live traffic flows. 🟡 Limits: only ingress is restricted, and only `portfolio` has policies of its own — see [Limits](#limits).

Picks up after [Alerting Reliability](./SRV-22-Alerting-Reliability.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the laptop node `<GPU_HOSTNAME>`, the admin user `<USER>`. Problems hit along the way: [SRV-28-TRBL](../troubleshooting/SRV-28-TRBL-Cilium-and-NetworkPolicy.md).

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Checks and tools](#step-1--checks-and-tools)
5. [Step 2 — Backup](#step-2--backup)
6. [Step 3 — Cilium as a second CNI](#step-3--cilium-as-a-second-cni)
7. [Step 4 — Migrate the nodes](#step-4--migrate-the-nodes)
8. [Step 5 — Finish and remove Flannel](#step-5--finish-and-remove-flannel)
9. [Step 6 — Prove that policies are enforced](#step-6--prove-that-policies-are-enforced)
10. [Step 7 — Hubble](#step-7--hubble)
11. [Step 8 — Policies for the portfolio](#step-8--policies-for-the-portfolio)
12. [Verification](#verification)
13. [Limits](#limits)
14. [Files](#files)
15. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Cilium 1.20.4 (Helm release `cilium`, chart `cilium/cilium`) | Namespace `kube-system`, DaemonSet on all three nodes | Pod networking (VXLAN tunnel) and `NetworkPolicy` enforcement in the kernel (eBPF) |
| `cilium-operator` | `kube-system`, one replica | Hands out pod CIDRs to nodes (cluster-pool IPAM) |
| `cilium-envoy` | `kube-system`, DaemonSet | Envoy for Cilium's L7 features; deployed by the chart, not used yet |
| Hubble relay and UI | `kube-system` | Live view of every flow and its verdict (forwarded or dropped) |
| `cilium` CLI, `hubble` CLI | Control plane, `/usr/local/bin/` | Install, status, flow observation |
| Pod CIDR `10.245.0.0/16` | Cilium | Replaces Flannel's `10.244.0.0/16` |
| Policies in `portfolio` (4) | Namespace `portfolio` | Default deny on ingress plus three allow rules |
| Kube-proxy | Unchanged | Cilium's kube-proxy replacement is not enabled |

## Why

The cluster ran Flannel, which carries pod traffic but **does not implement `NetworkPolicy`**. Policies could be created and were accepted by the API, yet nothing enforced them; the nine that already existed (six from ArgoCD, three from Strimzi) were decoration. That mattered for two reasons:

- The portfolio site, its API and its MongoDB are reachable from the internet through the public Gateway ([SRV-23](./SRV-23-Public-Gateway-and-HTTPS.md), [SRV-24](./SRV-24-Portfolio-Hosting.md)). Any pod in any namespace could still talk to the database directly.
- Network isolation is a standard Kubernetes security control and was the open item in [SRV-17](./SRV-17-Dedicated-GPU-Node.md) and [SRV-24](./SRV-24-Portfolio-Hosting.md).

A game server on the home network, planned next, is the third reason: it needs its own namespace with strict rules before it is opened to the internet.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Which CNI | Cilium | Enforces `NetworkPolicy` in eBPF, and brings Hubble (flow visibility) and L7 policy; Calico was the other candidate, with a documented live migration from Flannel too |
| Migration method | Cilium's documented node-by-node migration (`CiliumNodeConfig`) | Both CNIs run side by side with separate pod ranges; each node is switched on its own, so there is no cluster-wide cut-over |
| Pod CIDR | `10.245.0.0/16` | Must not overlap Flannel's `10.244.0.0/16`, the service range `10.96.0.0/12` or the node network |
| IPAM | `cluster-pool` | Required by the migration procedure |
| Overlay | VXLAN on UDP port **8473** | Flannel already uses the default 8472; the two overlays need different ports |
| Policy during migration | `policyEnforcementMode: never`, switched to `default` at the end | Nothing can be blocked while the cluster is half-migrated |
| Kube-proxy | Kept | Replacing it adds a second change to a cluster that is already being migrated |
| Order of nodes | Worker, laptop, control plane last | The control plane holds the API: its reboot is the only moment `kubectl` is unavailable |
| Version | Chart and images pinned to `1.20.4` | The CLI picks a default version; the chart is pinned explicitly so repeats are identical |
| Policy design | Observe with Hubble first, then write rules | Rules written from the real flows do not break dependencies nobody remembered |
| Starting scope | Ingress only, one namespace | Egress rules need DNS, API and external allow-lists; they are a later step |

## Step 1 — Checks and tools

Cilium needs a pod range that overlaps nothing, and a tunnel port that Flannel does not use:

```
kubectl cluster-info dump | grep -m2 -E 'cluster-cidr|service-cluster-ip-range'
kubectl -n kube-flannel get cm kube-flannel-cfg -o jsonpath='{.data.net-conf\.json}'; echo
ip -4 -br addr
sudo ufw status
uname -r
```

Pod CIDR `10.244.0.0/16`, service range `10.96.0.0/12`, Flannel on VXLAN with its default port. So `10.245.0.0/16` is free and port 8473 is used for Cilium. The firewall was inactive on all three nodes and the kernel (7.0) is far above Cilium's minimum.

The `cilium` CLI was installed from its GitHub release with a checksum check; the `hubble` CLI later the same way:

```
cd /tmp
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
```

## Step 2 — Backup

A change of CNI touches every node, so the nightly backup was run by hand first:

```
sudo ~/bin/backup-homelab.sh
ls -lt /var/backups/homelab | head -3
```

The script needs root: it reads `/etc/kubernetes/admin.conf` and the private keys under `pki`. Run without `sudo` it fails every critical step and leaves a `*-FAILED` folder ([SRV-28-TRBL](../troubleshooting/SRV-28-TRBL-Cilium-and-NetworkPolicy.md#backup-script-fails-with-permission-denied)). The expected result is `status: OK` with all five parts `ok`.

## Step 3 — Cilium as a second CNI

`platform/cilium/values-migration.yaml`:

```yaml
# Cilium in "secondary" mode, used only during the Flannel -> Cilium migration.
operator:
  unmanagedPodWatcher:
    restart: false          # do not restart pods that are not migrated yet
routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: 8473            # Flannel already uses 8472
cni:
  customConf: true          # do not write a CNI config file yet
  uninstall: false
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.245.0.0/16"]   # distinct from Flannel 10.244.0.0/16
policyEnforcementMode: "never"   # policies are enabled after the migration
bpf:
  hostLegacyRouting: true   # allow routing between Cilium and Flannel
```

The CLI expands this into the full set of Helm values (auto-detected settings), which are reviewed before installing:

```
cilium install --values values-migration.yaml --dry-run-helm-values > values-initial.yaml
cat values-initial.yaml

helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.20.4 \
  --namespace kube-system --values values-initial.yaml
cilium status --wait
```

All DaemonSets `3/3` and `Cluster Pods: 0/36 managed by Cilium`: the overlay is up, but no pod uses it yet. Then the object that decides which nodes switch. It matches no node until a node gets the label:

```
cat <<EOF | kubectl apply --server-side -f -
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: cilium-default
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
EOF
```

## Step 4 — Migrate the nodes

The same procedure for each node, with `NODE` set to its name. The worker goes first:

```
NODE=<WORKER_HOSTNAME>

kubectl cordon $NODE
kubectl drain --ignore-daemonsets --delete-emptydir-data $NODE

kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"

kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
```

Then a reboot of the node (`sudo reboot` on the node), so that every pod comes back on the new network. After it returns, a test pod pinned to the node must get an address from `10.245.0.0/16` and reach the API server:

```
kubectl -n kube-system run verify-network --restart=Never \
  --overrides='{"spec": {"nodeName": "'$NODE'", "tolerations": [{"operator": "Exists"}]}}' \
  --image ghcr.io/nicolaka/netshoot:v0.8 -- /bin/bash -c 'ip -br addr && curl -s -k https://$KUBERNETES_SERVICE_HOST/healthz && echo'
kubectl -n kube-system wait --for=jsonpath='{.status.phase}'=Succeeded pod/verify-network --timeout=120s
kubectl -n kube-system logs verify-network
kubectl -n kube-system delete pod verify-network

kubectl uncordon $NODE
```

Result: the pod address starts with `10.245.` and the last line is `ok`. The pod is started without `--attach` and read with `logs`: on a node that has not pulled the image yet, an attached run can end before it prints anything ([SRV-28-TRBL](../troubleshooting/SRV-28-TRBL-Cilium-and-NetworkPolicy.md#test-pod-shows-no-output-on-a-fresh-node)). Two checks after the worker, before the next node: no pod outside `Running` or `Completed`, and the Kafka consumer lag still at `0`. The worker holds the whole Kafka and monitoring stack, so its drain stops the log pipeline for a few minutes ([SRV-28-TRBL](../troubleshooting/SRV-28-TRBL-Cilium-and-NetworkPolicy.md#kafka-exporter-and-entity-operator-crash-after-the-worker-drain)).

**The laptop** follows the same steps. Its graphical session blocks a plain reboot; `sudo systemctl reboot -i` ignores the inhibitor and closes the session ([SRV-28-TRBL](../troubleshooting/SRV-28-TRBL-Cilium-and-NetworkPolicy.md#reboot-of-the-laptop-is-inhibited)). Only system DaemonSet pods run there (the node is tainted for AI workloads), so the drain is instant.

**The control plane** goes last. It runs both CoreDNS pods, still on the old network. A plain drain would take cluster DNS down for the time it takes them to start elsewhere, so the node is cordoned first and CoreDNS is restarted onto the migrated nodes before the drain:

```
NODE=<HOSTNAME>

kubectl cordon $NODE
kubectl -n kube-system rollout restart deploy/coredns
kubectl -n kube-system rollout status deploy/coredns
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide     # both on migrated nodes, 10.245.x.x

kubectl drain --ignore-daemonsets --delete-emptydir-data $NODE
kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
sudo reboot
```

The API server is unavailable for two to three minutes during the reboot; running pods on the other nodes are not affected. The dead man's switch ([SRV-22](./SRV-22-Alerting-Reliability.md)) runs on this node, so it cannot report its own reboot. After the node returns, the same test pod and `uncordon` as above. `cilium status` then shows `Cluster Pods: 36/36 managed by Cilium`.

## Step 5 — Finish and remove Flannel

Cilium becomes the primary CNI and policy enforcement is switched on. The CLI generates the final values and the difference is reviewed:

```
cilium install --values values-initial.yaml --dry-run-helm-values \
  --set operator.unmanagedPodWatcher.restart=true \
  --set cni.customConf=false \
  --set policyEnforcementMode=default > values-final.yaml

diff values-initial.yaml values-final.yaml
```

Exactly three lines change: `customConf`, `restart` and `policyEnforcementMode`. eBPF host routing (`bpf.hostLegacyRouting=false`) is an optional extra that interrupts each node briefly and was left off.

```
helm upgrade --namespace kube-system cilium cilium/cilium --version 1.20.4 --values values-final.yaml
kubectl -n kube-system rollout restart daemonset cilium
cilium status --wait
```

From this moment the nine existing policies (ArgoCD, Strimzi) are enforced for the first time, so the first things to check are what they could have broken: pods outside `Running`, the `alloy-consumer` lag, and the ArgoCD applications:

```
kubectl get pods -A | grep -v -E 'Running|Completed'
kubectl -n kafka exec homelab-dual-role-0 -- bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 --describe --group alloy-consumer
kubectl -n argocd get applications
```

Result: no problem pods once one MetalLB pod was restarted ([SRV-28-TRBL](../troubleshooting/SRV-28-TRBL-Cilium-and-NetworkPolicy.md#metallb-statuscleaner-in-crashloopbackoff)), lag `0` to `3`, both applications `Synced` and `Healthy`. The policies as written did not need any change.

Then the temporary object, and Flannel itself, whose manifests are kept for a rollback:

```
kubectl delete -n kube-system ciliumnodeconfig cilium-default
cilium status | grep 'Cluster Pods'             # N/N managed by Cilium before going on

cd platform/cilium/flannel-backup
kubectl -n kube-flannel get ds,cm,sa -o yaml > flannel-namespaced.yaml
kubectl get clusterrole flannel -o yaml > flannel-clusterrole.yaml
kubectl get clusterrolebinding flannel -o yaml > flannel-clusterrolebinding.yaml

kubectl delete namespace kube-flannel
kubectl delete clusterrole flannel
kubectl delete clusterrolebinding flannel
```

Flannel may leave a CNI config file (`/etc/cni/net.d/10-flannel.conflist`), the `flannel.1` and `cni0` interfaces and some firewall rules on each node; Cilium's own config file sorts first and takes precedence, and the rest disappears at the next reboot of each node.

## Step 6 — Prove that policies are enforced

The point of the migration, tested in a throw-away namespace: a web server answers, then a default-deny policy is added and the same request must fail.

```
kubectl create namespace np-test
kubectl -n np-test run web --image=nginx:alpine --port=80
kubectl -n np-test expose pod web --port=80
kubectl -n np-test wait --for=condition=Ready pod/web --timeout=90s

kubectl -n np-test run c1 --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s -o /dev/null -m 5 -w '%{http_code}\n' http://web           # 200

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: np-test
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

kubectl -n np-test run c2 --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s -o /dev/null -m 5 -w '%{http_code}\n' http://web           # 000

kubectl delete namespace np-test
```

`200` before the policy, `000` (timeout) after. With Flannel the second request would have returned `200`.

## Step 7 — Hubble

Policies are written from observed traffic, not guessed. Hubble is switched on with a separate values file, so the change is reproducible:

```yaml
# platform/cilium/values-hubble.yaml
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
```

```
helm upgrade --namespace kube-system cilium cilium/cilium --version 1.20.4 \
  -f values-final.yaml -f values-hubble.yaml
cilium status --wait
```

Two ways to look at the flows. The command line, with the `hubble` CLI installed from its release the same way as the `cilium` CLI:

```
cilium hubble port-forward &
hubble observe --namespace kafka --last 20
hubble observe --namespace portfolio --verdict DROPPED --last 20 -o compact
```

And the web interface, which is only reachable inside the cluster, so the port is forwarded and tunnelled to the workstation:

```
kubectl -n kube-system port-forward svc/hubble-ui 12000:80          # on the control plane
ssh -L 12000:localhost:12000 <USER>@<SERVER_IP>                     # on the workstation, then http://localhost:12000
```

The first `hubble observe` already shows the real dependencies of the Kafka namespace: both Alloy releases talk to the broker on `9092`, the entity operator on `9091`, Prometheus scrapes the operators' metrics, and kubelet probes arrive from the node and are allowed by Cilium by default.

## Step 8 — Policies for the portfolio

The public site is the one place where the internet reaches workloads, so it came first. Hubble showed exactly three flows:

| From | To | Port |
|---|---|---|
| Envoy Gateway (namespace `envoy-gateway-system`) | `web` | 8080 |
| `web` | `api` | 5050 |
| `api` | `mongo` | 27017 |

`platform/network-policies/portfolio.yaml` holds one default-deny policy and one allow rule per flow:

```yaml
# Ingress policy for the public portfolio site.
# Allowed flows (observed with Hubble):
#   envoy-gateway-system -> web:8080 -> api:5050 -> mongo:27017
# kubelet probes come from the node (host) and are always allowed by Cilium.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: portfolio
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gateway-to-web
  namespace: portfolio
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: envoy-gateway-system
      ports:
        - port: 8080
          protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: portfolio
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - port: 5050
          protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-mongo
  namespace: portfolio
spec:
  podSelector:
    matchLabels:
      app: mongo
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 27017
          protocol: TCP
```

Validated without applying, then applied and tested from a pod in another namespace:

```
kubectl apply -f platform/network-policies/portfolio.yaml --dry-run=server
kubectl apply -f platform/network-policies/portfolio.yaml

kubectl -n default run t1 --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s -o /dev/null -m 5 -w 'mongo from default: %{http_code}\n' http://mongo.portfolio:27017
kubectl -n default run t2 --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s -o /dev/null -m 5 -w 'api from default: %{http_code}\n' http://api.portfolio:5050

hubble observe --namespace portfolio --verdict DROPPED --last 10 -o compact
```

Both requests returned `000`, and Hubble listed only the test pods as `INGRESS DENIED`. The public site, opened from a phone on mobile data, kept working, including the pages that call the API. The rollback if a page ever breaks is one command: `kubectl -n portfolio delete networkpolicy --all`.

## Verification

```
cilium status
kubectl get pods -A | grep -v -E 'Running|Completed'
kubectl -n kube-system get pods | grep hubble
kubectl -n portfolio get networkpolicy
```

`cilium status` shows every component ready and all pods managed by Cilium; the problem list is empty; `hubble-relay` and `hubble-ui` run; four policies in `portfolio`. The log pipeline ([SRV-21](./SRV-21-Kafka-Strimzi-Log-Pipeline.md)) and the public site work as before.

## Limits

- **Only ingress is restricted.** Pods may still open any outgoing connection. Egress rules need allow-lists for DNS, the API server and external services, and are a separate step.
- **One namespace has its own policies.** Besides `portfolio`, only the ArgoCD and Strimzi policies exist; `monitoring`, `envoy-gateway-system`, `cert-manager` and the rest still accept any ingress. They are added one namespace at a time, each after looking at its flows in Hubble.
- **The policies are applied by hand.** `platform/network-policies/` is in Git, but ArgoCD does not watch that path yet; the portfolio application manages the namespace's workloads, not these files.
- **Hubble has no persistent history.** Flows are kept in a small in-memory ring buffer per node; a flow from last week cannot be looked up.
- **The web interface is not published.** Hubble UI is reached with a port-forward and an SSH tunnel on purpose; it shows the whole cluster's traffic.
- **Flannel leftovers stay until each node reboots.** A config file, two interfaces and some firewall rules are harmless but present.
- **Kube-proxy is still in use.** Cilium could replace it; that was left for a separate change.
- **The control plane is a single point during its reboot.** For two to three minutes there is no API and no dead man's switch; this is inherent in a single control-plane node.

## Files

| File | Location | Contents |
|---|---|---|
| `values-migration.yaml` | `platform/cilium/` in `homelab-gitops` | Values for the temporary second-CNI mode |
| `values-initial.yaml` | `platform/cilium/` | Full values generated from the previous file, used for the install |
| `values-final.yaml` | `platform/cilium/` | Final values: Cilium primary, policies on |
| `values-hubble.yaml` | `platform/cilium/` | Hubble relay and UI |
| `flannel-backup/` | `platform/cilium/` | The removed Flannel objects, for a rollback |
| `portfolio.yaml` | `platform/network-policies/` | Default deny and three allow rules for `portfolio` |

## Related

- [Public Gateway and HTTPS](./SRV-23-Public-Gateway-and-HTTPS.md) and [Portfolio Hosting](./SRV-24-Portfolio-Hosting.md) — what the portfolio policies protect
- [Kafka with Strimzi and a Log Pipeline](./SRV-21-Kafka-Strimzi-Log-Pipeline.md) — the workload most affected by the migration
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — where Flannel was installed
- [Cilium: Migrating a cluster to Cilium](https://docs.cilium.io/en/stable/installation/k8s-install-migration/) — the procedure followed here
