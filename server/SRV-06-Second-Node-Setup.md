---
tags: [homelab, project, os, second-node]
---

# Second Node Setup — Dell OptiPlex 7050 (Worker)

> Status: 🟢 **Joined and Ready.** Cluster is now two nodes: `<HOSTNAME>` (control-plane) and `<WORKER_HOSTNAME>` (worker). See [Hardware Selection](./SRV-01-Hardware-Selection.md) for how this machine was chosen and [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) for the control-plane side this node joined.

Installed on: **Dell OptiPlex 7050 Micro** (second/worker node)

## Hardware recap

- Intel Core i7-6700T (4 cores / 8 threads via Hyper-Threading, 2.8GHz base, up to 3.6GHz boost)
- 16GB DDR4 RAM (included)
- 512GB SSD
- Purchased used/refurbished from seller **iBankonIT, LLC** via eBay, 90-day warranty
- Shipped with Windows 10/11 Pro preinstalled — wiped in favor of Ubuntu Server

Full comparison against the OptiPlex 3050 (passed over) is in [Hardware Selection](./SRV-01-Hardware-Selection.md).

## Installation media

Same process as the primary node — see [OS Installation](./SRV-02-OS-Installation.md) for the full Rufus/ISO details (Ubuntu Server 26.04 LTS, GPT/UEFI, ISO Image mode). Not repeated here.

## Dell-specific BIOS keys

This Dell hardware uses different boot/BIOS keys than the GMKtec M8 primary node (which was never explicitly recorded — worth noting for next time, since it's easy to assume all machines share the same key):

| Key | Action |
|---|---|
| **F2** | Enter BIOS/UEFI Setup |
| **F12** | One-time Boot Menu (select the installer USB without changing permanent boot order) |

## Installer walkthrough (key choices made)

| Step | Choice |
|---|---|
| Base install | **Ubuntu Server** (not "minimized") |
| Third-party drivers | Skipped |
| Network configuration | Skipped at install time — to be configured post-install, same as the primary node |
| Storage layout | Guided, entire disk, **LVM enabled**, **no LUKS encryption** |
| Hostname | `<WORKER_HOSTNAME>` |
| Username | `<USERNAME>` (same as the primary node, for consistency across the cluster) |
| SSH | **OpenSSH server installed**, password authentication over SSH allowed |

## Known recurring issue: LVM under-allocates the root partition

The same LVM behavior documented on the primary node (see [OS Installation](./SRV-02-OS-Installation.md)) showed up again here. Confirmed via `fastfetch`:
```
Disk (/): 6.92 GiB / 97.87 GiB (7%)
```
despite the physical drive being 512GB — the guided installer only assigned ~100GB to the root logical volume and left the rest of the disk as unallocated free space in the volume group. This is the installer's default behavior, not something specific to this hardware — see [LVM Partition Sizing](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-LVM-Partition-Sizing.md) for the full explanation and the standard `lvextend` + `resize2fs` fix.

**Status: ⬜ planned, not yet executed on this node.**

## Network configuration

Connected via Ethernet to the switch (Port 5, VLAN 20 — see [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md)). Unlike the primary node's static-IP setup ([Network Configuration](./SRV-03-Network-Configuration.md)), this node was left on DHCP — both an interim choice made while diagnosing the issues below, and, since a worker node doesn't need a fixed address for `kubeadm join` to succeed (only the control-plane's address matters for that), never revisited afterward. **Open item:** consider a DHCP reservation for `<WORKER_IP>` for long-term stability, consistent with the fixed-address treatment given to other Admin-VLAN infrastructure.

### Real troubleshooting: no IPv4 address despite a healthy link

**Symptom:** `ip a` showed the Ethernet interface (`enp0s31f6`) as `UP` with `LOWER_UP` set (confirming a live physical link to the switch) and a normal IPv6 link-local address, but no IPv4 address at all.

**Two separate causes, found in sequence:**

1. **Switch port PVID mismatch** — the same class of bug documented in [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md): Port 5's VLAN membership looked correct, but its PVID was still the default `1`. Fixed via the switch's `802.1Q PVID Setting` page (`Port 5 → PVID 20`).

2. **netplan config had no `dhcp4` directive at all.** Even after the PVID fix, no IPv4 address was requested. Inspecting the installer-generated config:

   ```
   sudo cat /etc/netplan/*.yaml
   ```

   ```yaml
   network:
     ethernets:
       enp0s31f6:
         match:
           macaddress: <WORKER_MAC>
         set-name: enp0s31f6
     version: 2
   ```

   The same root cause already documented on the primary node ([Network Configuration, "Correct config"](./SRV-03-Network-Configuration.md)): the installer's `match`/`set-name` block only renames the interface — it never specifies `dhcp4` or a static address. **Fix:**

   ```
   sudo nano /etc/netplan/00-installer-config.yaml
   ```

   ```yaml
   network:
     ethernets:
       enp0s31f6:
         match:
           macaddress: <WORKER_MAC>
         set-name: enp0s31f6
         dhcp4: true
     version: 2
   ```

   ```
   sudo netplan apply
   ```

   The node obtained `<WORKER_IP>` immediately afterward.

## Docker installation

Same rationale and packages as the primary node — see [Docker Installation](./SRV-04-Docker-Installation.md) for why Docker is installed alongside (not instead of) containerd:

```
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose-v2 git curl wget htop fastfetch -y
sudo usermod -aG docker $USER
exit
ssh <USERNAME>@<WORKER_IP>
groups                  # confirm 'docker' appears
docker run hello-world
```

## Kubernetes prerequisites

Identical to the primary node — see [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) for the full explanation of each step. Commands run on this node:

```
# Swap
sudo nano /etc/fstab          # comment out the swap line
sudo swapoff -a

# containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Kernel modules and sysctl — br_netfilter made persistent immediately this time
sudo modprobe overlay
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

# kubeadm / kubelet / kubectl — same v1.31 repo as the control-plane node
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Preflight dependency
sudo apt install -y conntrack ethtool socat
```

**Note on `br_netfilter`:** registering it in `/etc/modules-load.d/k8s.conf` immediately, rather than only after hitting a failure, was a direct lesson from [Kubernetes Installation, Step 8](./SRV-05-Kubernetes-Installation.md#step-8--installing-flannel-cni), where the same module silently dropped across a reboot and crashed Flannel on the primary node. Applying the fix proactively here meant this node reached `Ready` without repeating that incident.

## Real troubleshooting: `kubeadm init` run by mistake instead of `kubeadm join`

**What happened:** the command actually run on this node was:

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

This is the command for bootstrapping a **new, first** control-plane node — not for joining an existing cluster. It completed "successfully," which was itself misleading: the output was shaped identically to the original control-plane bootstrap in [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md), including its own fresh `kubeadm join ...` line at the end, generated from this node's own new (and unwanted) control plane.

**Root cause:** `kubeadm init` and `kubeadm join` are different operations invoked through the same `sudo kubeadm <verb>` shape, and `kubeadm` has no way to know a given machine was only ever intended to be a worker — "create a new cluster here" is a perfectly valid, well-formed request from its point of view. See [Guide: kubeadm init vs. kubeadm join](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Kubeadm-Init-vs-Join.md) for the general pattern.

**Consequence:** this node briefly became the control-plane of its **own, separate, independent** Kubernetes cluster — its own CA, its own etcd, its own API server on `<WORKER_IP>:6443` — entirely disconnected from the real cluster on `<SERVER_IP>`.

**Fix — reset the node to a clean slate:**

```
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
```

- `/etc/cni/net.d` — stale Flannel config from the accidental `init`, which could otherwise confuse the real join
- `$HOME/.kube` — kubeconfig pointing at the now-deleted throwaway cluster; irrelevant for a worker, which never needs its own admin credentials
- the `iptables` commands flush and remove the custom chains `kube-proxy` had installed for the fake cluster

**Generated a fresh join command from the real control plane** (run on `<HOSTNAME>`, not this node):

```
kubeadm token create --print-join-command
```

The original token from the primary node's `kubeadm init` (see [Kubernetes Installation, Step 6](./SRV-05-Kubernetes-Installation.md#step-6--kubeadm-init)) had already expired — bootstrap tokens default to a 24-hour lifetime.

**Joined correctly** (run on this node, this time with the right verb):

```
sudo kubeadm join <SERVER_IP>:6443 --token <JOIN_TOKEN> \
        --discovery-token-ca-cert-hash sha256:<CA_CERT_HASH>
```

**Note on `kubectl` access:** unlike `kubeadm init`, `kubeadm join` produces no `admin.conf` — a worker has no need for cluster-admin credentials of its own. The `~/.kube/config` copy step from [Kubernetes Installation, Step 7](./SRV-05-Kubernetes-Installation.md#step-7--kubectl-access) is not repeated on this node; all `kubectl` commands continue to run from `<HOSTNAME>`.

## Verification

From the control-plane node:

```
kubectl get nodes
```

```
NAME                 STATUS   ROLES           AGE   VERSION
<HOSTNAME>           Ready    control-plane   14d   v1.31.14
<WORKER_HOSTNAME>    Ready    <none>          14s   v1.31.14
```

Confirmed real scheduling works, not just node registration:

```
kubectl create deployment nginx-test --image=nginx
kubectl get pods -o wide     # confirmed the pod landed on <WORKER_HOSTNAME>, not the control-plane node
kubectl delete deployment nginx-test
```

This is expected now without needing to temporarily lift the control-plane's `NoSchedule` taint (see [Kubernetes Installation, Step 10](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint)) — a real worker now exists to absorb ordinary workloads, so the taint stays on the control-plane permanently going forward.

## Related

- [Hardware Selection](./SRV-01-Hardware-Selection.md) — why the OptiPlex 7050 was chosen over the 3050
- [OS Installation](./SRV-02-OS-Installation.md) — full install-media process, shared with the primary node
- [Network Configuration](./SRV-03-Network-Configuration.md) — the primary node's netplan setup, whose missing-`dhcp4` bug recurred here
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — control-plane side this node joined
- [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md) — the PVID bug that also affected this node's switch port
- [LVM Partition Sizing](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-LVM-Partition-Sizing.md) — companion Guides repository — the root-partition issue, still unresolved on this node
- [Guide: kubeadm init vs. kubeadm join](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Kubeadm-Init-vs-Join.md) — companion Guides repository, written directly from the mistake documented above
- [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
