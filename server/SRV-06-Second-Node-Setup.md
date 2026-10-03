---
tags: [homelab, project, os, second-node]
---

# Second Node Setup — Dell OptiPlex 7050 (Worker)

> Status: 🟢 **Joined and Ready.** Cluster is now two nodes: `<HOSTNAME>` (control-plane) and `<WORKER_HOSTNAME>` (worker). See [Hardware Selection](./SRV-01-Hardware-Selection.md) for how this machine was chosen and [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) for the control-plane side this node joined.

Installed on: **Dell OptiPlex 7050 Micro** (second/worker node)

## Contents

1. [Hardware recap](#hardware-recap)
2. [Installation media](#installation-media)
3. [Dell-specific BIOS keys](#dell-specific-bios-keys)
4. [Installer walkthrough (key choices made)](#installer-walkthrough-key-choices-made)
5. [Known recurring issue: LVM under-allocates the root partition](#known-recurring-issue-lvm-under-allocates-the-root-partition)
6. [Network configuration](#network-configuration)
7. [Docker installation](#docker-installation)
8. [Kubernetes prerequisites](#kubernetes-prerequisites)
9. [Joining the cluster](#joining-the-cluster)
10. [Verification](#verification)
11. [Related](#related)

Problems hit during this work are recorded separately in [SRV-06-TRBL](../troubleshooting/SRV-06-TRBL-Second-Node-Setup.md).

Components installed on this node match the primary node (Ubuntu Server 26.04, Docker, containerd, kubelet/kubeadm/kubectl v1.31); see [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md#components-installed).

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

Differ from the GMKtec M8 (keys for that machine were not recorded):

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

Same as the primary node ([OS Installation](./SRV-02-OS-Installation.md)). `fastfetch` on a 512GB drive:
```
Disk (/): 6.92 GiB / 97.87 GiB (7%)
```
Guided installer default — ~100GB root LV, remainder unallocated in the VG. See [LVM Partition Sizing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-LVM-Partition-Sizing.md).

**Status: ✅ fixed.**

```
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Result: `df -h /` 466G available; `vgs` `VFree 0`. Primary node fixed later — see [Cluster Verification](./SRV-07-Cluster-Verification.md).

## Network configuration

Ethernet to switch Port 5, VLAN 20, port PVID 20 ([VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md)). Static IP `<WORKER_IP>`, confirmed in [Cluster Verification](./SRV-07-Cluster-Verification.md).

The installer-written netplan file contains only `match`/`set-name`, as on the primary node ([Network Configuration](./SRV-03-Network-Configuration.md)); addressing is added to it:

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
      dhcp4: no
      addresses:
        - <WORKER_IP>/24
      routes:
        - to: default
          via: <GATEWAY_IP>
      nameservers:
        addresses:
          - <GATEWAY_IP>
  version: 2
```

```
sudo netplan apply
```

`nameservers` → router, not public resolvers ([Network Configuration — DNS](./SRV-03-Network-Configuration.md#dns)).

The node first came up with no IPv4 address and ran on DHCP before the static config — see [SRV-06-TRBL](../troubleshooting/SRV-06-TRBL-Second-Node-Setup.md#no-ipv4-address-despite-a-healthy-link).

## Docker installation

Same packages as the primary node ([Docker Installation](./SRV-04-Docker-Installation.md)):

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

Same steps as [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md):

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

`br_netfilter` registered in `/etc/modules-load.d/k8s.conf` up front, avoiding the reboot-persistence failure hit on the primary node ([SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md#flannel-in-crashloopbackoff-br_netfilter-not-loaded)).

## Joining the cluster

A worker joins with `kubeadm join`, never `kubeadm init`. Join command generated on `<HOSTNAME>` (the token printed by the original `kubeadm init` had expired — 24h default):

```
kubeadm token create --print-join-command
```

On this node:

```
sudo kubeadm join <SERVER_IP>:6443 --token <JOIN_TOKEN> \
        --discovery-token-ca-cert-hash sha256:<CA_CERT_HASH>
```

No kubeconfig on the worker (`kubeadm join` produces no `admin.conf`); `kubectl` runs from `<HOSTNAME>`.

`kubeadm init` was first run on this node by mistake and the node had to be reset before joining — see [SRV-06-TRBL](../troubleshooting/SRV-06-TRBL-Second-Node-Setup.md#kubeadm-init-run-instead-of-kubeadm-join).

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

Scheduling test:

```
kubectl create deployment nginx-test --image=nginx
kubectl get pods -o wide     # pod on <WORKER_HOSTNAME>
kubectl delete deployment nginx-test
```

Scheduled on the worker with the control-plane taint in place; the taint remains permanently.

## Related

- [SRV-06-TRBL](../troubleshooting/SRV-06-TRBL-Second-Node-Setup.md) — troubleshooting for this doc
- [Hardware Selection](./SRV-01-Hardware-Selection.md) — why the OptiPlex 7050 was chosen over the 3050
- [OS Installation](./SRV-02-OS-Installation.md) — full install-media process, shared with the primary node
- [Network Configuration](./SRV-03-Network-Configuration.md) — the primary node's netplan setup, whose installer-written netplan file has the same gap
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — control-plane side this node joined
- [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md) — switch port and PVID for this node
- [LVM Partition Sizing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-LVM-Partition-Sizing.md) — companion Guides repository — the root-partition issue, fixed on this node
- [Cluster Verification](./SRV-07-Cluster-Verification.md) — live check of both nodes against these docs
- [Guide: kubeadm init vs. kubeadm join](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubeadm-Init-vs-Join.md) — companion Guides repository, written from the `init`-instead-of-`join` mistake
- [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
