---
tags: [homelab-project, homelab, note, kubernetes, project]
---

# Kubernetes Installation (kubeadm)

> Status: 🟡 **In progress**. All prerequisites installed. `kubeadm init` has not yet been run at time of writing.

## Decision: kubeadm over k3s

The project initially considered **k3s** (a lightweight Kubernetes distribution) for its simplicity on constrained hardware. This was reconsidered in favor of full **kubeadm-based Kubernetes**, prioritizing closer-to-production learning value over ease of setup. See [Kubernetes vs k3s](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Kubernetes-vs-k3s.md) for the full comparison.

Note: k3s was never actually installed in this environment — an attempted uninstall (`/usr/local/bin/k3s-uninstall.sh`) returned "command not found," confirming no prior installation existed. No conflict with the kubeadm path.

## Step 1 — Disable swap

Kubernetes' default kubelet configuration (`NoSwap` behavior) requires swap to be off. See [Kubernetes Swap Requirement](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Kubernetes-Swap-Requirement.md) for why this is still the default even in recent Kubernetes releases with experimental swap support.

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

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

- `overlay` — required for the container image layer filesystem
- `br_netfilter` — lets bridged network traffic between pods be processed by iptables
- `ip_forward = 1` — enables packet forwarding between interfaces; required for pod-to-pod and pod-to-internet traffic. This is a local, internal mechanism, not an externally-facing security hole by itself — see [Homelab Network Security](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Homelab-Network-Security.md).

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

## Next step (not yet executed)

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

This will:
1. Bootstrap the control plane on `<HOSTNAME>` (<SERVER_IP>)
2. Output a `kubeadm join` command with a token — **must be saved**, needed later to join the OptiPlex node
3. Require post-init kubeconfig setup:
   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
4. A CNI plugin (Flannel, matching the `10.244.0.0/16` CIDR) must be installed afterward — the node will show `NotReady` until this is done.

## Related

- [Kubernetes vs k3s](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Kubernetes-vs-k3s.md) — companion Guides repository
- [Docker vs Containerd](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Docker-vs-Containerd.md) — companion Guides repository
- [Kubernetes Swap Requirement](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Kubernetes-Swap-Requirement.md) — companion Guides repository
- [Linux Fundamentals](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Linux-Fundamentals.md) — companion Guides repository — explains `/etc`, `sudo`, `modprobe`, `sed` used throughout this process
