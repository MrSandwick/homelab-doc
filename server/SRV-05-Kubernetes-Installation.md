---
tags: [homelab, project, kubernetes]
---

# Kubernetes Installation (kubeadm)

> Status: 🟢 **Control plane initialized and healthy.** CNI (Flannel) installed. First test workload deployed and verified. Second node has since joined — see [Second Node Setup](./SRV-06-Second-Node-Setup.md).

## Contents

1. [Components installed](#components-installed)
2. [Decision: kubeadm over k3s](#decision-kubeadm-over-k3s)
3. [Step 1 — Disable swap](#step-1--disable-swap)
4. [Step 2 — Install and configure containerd](#step-2--install-and-configure-containerd)
5. [Step 3 — Kernel modules and sysctl settings](#step-3--kernel-modules-and-sysctl-settings)
6. [Step 4 — Install kubeadm, kubelet, kubectl](#step-4--install-kubeadm-kubelet-kubectl)
7. [Step 5 — Preflight dependencies](#step-5--preflight-dependencies)
8. [Step 6 — `kubeadm init`](#step-6--kubeadm-init)
9. [Step 7 — kubectl access](#step-7--kubectl-access)
10. [Step 8 — Installing Flannel (CNI)](#step-8--installing-flannel-cni)
11. [Step 9 — Node reaches `Ready`](#step-9--node-reaches-ready)
12. [Step 10 — First test workload and the control-plane taint](#step-10--first-test-workload-and-the-control-plane-taint)
13. [Related](#related)

Problems hit during this work are recorded separately in [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md).

## Components installed

| Component | Version / source | Role |
|---|---|---|
| containerd | Ubuntu `containerd` package | Container runtime used by the kubelet (`SystemdCgroup = true`) |
| kubelet, kubeadm, kubectl | `pkgs.k8s.io` v1.31, held with `apt-mark hold` | Node agent, cluster bootstrap, CLI |
| conntrack, ethtool, socat | Ubuntu packages | `kubeadm` preflight dependencies |
| Flannel | `kube-flannel.yml` (latest release) | CNI — pod network `10.244.0.0/16` |

Overview of each component: [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md).

## Decision: kubeadm over k3s

The project initially considered **k3s** (a lightweight Kubernetes distribution) for its simplicity on constrained hardware. This was reconsidered in favor of full **kubeadm-based Kubernetes**, prioritizing closer-to-production learning value over ease of setup. See [Kubernetes vs k3s](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-vs-k3s.md) for the full comparison.

Note: k3s was never actually installed in this environment — an attempted uninstall (`/usr/local/bin/k3s-uninstall.sh`) returned "command not found," confirming no prior installation existed. No conflict with the kubeadm path.

## Step 1 — Disable swap

Kubernetes' default kubelet configuration (`NoSwap` behavior) requires swap to be off. See [Kubernetes Swap Requirement](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Swap-Requirement.md) for why this is still the default even in recent Kubernetes releases with experimental swap support.

Chosen approach: **disable without deleting** the swap file, by commenting out its line in `/etc/fstab` rather than removing it — so it can be re-enabled later if needed.

```
sudo nano /etc/fstab
# comment out the line containing "swap", e.g.:
# /swap.img none swap sw 0 0

sudo swapoff -a
free -h   # confirm Swap shows 0
```

To re-enable in the future: uncomment the line and run `sudo swapon -a`.

## Step 2 — Install and configure containerd

```
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

The `SystemdCgroup = true` change is mandatory — containerd and kubelet must use the same cgroup driver (systemd) or the kubelet will fail to register the node correctly.

Verify:
```
sudo systemctl status containerd
```

## Step 3 — Kernel modules and sysctl settings

```
sudo modprobe overlay
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

- `overlay` — required for the container image layer filesystem
- `br_netfilter` — lets bridged network traffic between pods be processed by iptables
- `/etc/modules-load.d/k8s.conf` — registers `br_netfilter` for autoload; `modprobe` alone does not survive a reboot. Added after the module was lost on reboot — see [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md#flannel-in-crashloopbackoff-br_netfilter-not-loaded)
- `ip_forward = 1` — enables packet forwarding between interfaces; required for pod-to-pod and pod-to-internet traffic. This is a local, internal mechanism, not an externally-facing security hole by itself — see [Homelab Network Security](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/host/Guide-Homelab-Network-Security.md).

## Step 4 — Install kubeadm, kubelet, kubectl

```
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

- **kubelet** — the per-node agent that actually manages containers according to the cluster's instructions
- **kubeadm** — the tool used to bootstrap the cluster (`kubeadm init`, `kubeadm join`)
- **kubectl** — the CLI used to interact with the cluster afterward
- `apt-mark hold` prevents `apt upgrade` from silently changing Kubernetes versions, which can break a cluster

Verify:
```
kubeadm version
kubectl version --client
```

## Step 5 — Preflight dependencies

`kubeadm init` preflight requires `conntrack` (used by `kube-proxy` to track connection state in netfilter); installed along with two other commonly-required tools:
```
sudo apt update
sudo apt install -y conntrack ethtool socat
```

Found missing on the first `kubeadm init` attempt — see [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md#kubeadm-init-preflight-conntrack-not-found).

## Step 6 — `kubeadm init`

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Output (abridged):
```
Your Kubernetes control-plane has initialized successfully!
...
kubeadm join <SERVER_IP>:6443 --token <JOIN_TOKEN> \
        --discovery-token-ca-cert-hash sha256:<CA_CERT_HASH>
```

**Note on the join token:** bootstrap tokens expire after 24 hours by default. Since the second node (OptiPlex) was not ready to join at the time, the original token was allowed to expire rather than being preserved. When the second node is ready, a fresh token is generated with:
```
kubeadm token create --print-join-command
```
This prints a complete, ready-to-run `kubeadm join` command with a new token and current CA cert hash — no need to reuse the original one from `kubeadm init`.

## Step 7 — kubectl access

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verified with:
```
kubectl get nodes
```
Result: one node, status `NotReady` (expected — no CNI installed yet).

## Step 8 — Installing Flannel (CNI)

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel runs in its own `kube-flannel` namespace, not `kube-system`. It requires the `br_netfilter` module from Step 3 to be loaded; on the first attempt the pod crash-looped because the module had been lost on reboot — see [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md#flannel-in-crashloopbackoff-br_netfilter-not-loaded).

## Step 9 — Node reaches `Ready`

```
kubectl get nodes
```
Status changed from `NotReady` to **`Ready`** once Flannel was healthy and CoreDNS pods (previously stuck in `ContainerCreating`, waiting on pod networking) transitioned to `Running`.

## Step 10 — First test workload and the control-plane taint

A throwaway nginx deployment confirms the cluster works end-to-end. `kubeadm init` taints the control-plane node (`node-role.kubernetes.io/control-plane:NoSchedule`), so on a single-node cluster the taint is removed for the duration of the test (the trailing `-` removes it):
```
kubectl taint nodes <HOSTNAME> node-role.kubernetes.io/control-plane:NoSchedule-
kubectl create deployment nginx-test --image=nginx
kubectl get pods
```

**Verification:**
```
kubectl port-forward --address 0.0.0.0 deployment/nginx-test 8080:80
```
then, from another machine on the LAN, `http://<SERVER_IP>:8080` served the default nginx welcome page — confirming pod scheduling, pod networking (Flannel), and port-forwarding all work correctly. `--address 0.0.0.0` is required for LAN access; `port-forward` binds to `localhost` only by default.

Incidents in this step (pod `Pending` before the taint was removed; `port-forward` unreachable): [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md#test-pod-stuck-pending-control-plane-taint).

**Cleanup — removing the test workload and restoring the taint:**
```
kubectl delete deployment nginx-test
kubectl taint nodes <HOSTNAME> node-role.kubernetes.io/control-plane:NoSchedule
```
Verified the taint was restored:
```
kubectl describe node <HOSTNAME> | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

This taint stays on the control-plane permanently from here on: the OptiPlex worker node has since joined the cluster and absorbs ordinary workloads instead, so there's no longer any reason to lift it. See [Second Node Setup](./SRV-06-Second-Node-Setup.md).

## Next step — completed

The second node joined successfully. Full process in [Second Node Setup](./SRV-06-Second-Node-Setup.md).

## Related

- [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md) — troubleshooting for this doc
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the OptiPlex worker node that has since joined this cluster
- [Guide: kubeadm init vs. kubeadm join](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubeadm-Init-vs-Join.md) — companion Guides repository
- [Kubernetes vs k3s](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-vs-k3s.md) — companion Guides repository
- [Docker vs Containerd](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Docker-vs-Containerd.md) — companion Guides repository
- [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
- [Pod Networking and CNI](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Pod-Networking-and-CNI.md) — companion Guides repository
- [Kubernetes Swap Requirement](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Swap-Requirement.md) — companion Guides repository
- [Linux Fundamentals](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/host/Guide-Linux-Fundamentals.md) — companion Guides repository — explains `/etc`, `sudo`, `modprobe`, `sed` used throughout this process
