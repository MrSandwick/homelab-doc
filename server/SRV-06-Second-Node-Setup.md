---
tags: [homelab, project, os, second-node]
---

# Second Node Setup — Dell OptiPlex 7050 (Worker)

> Status: 🟡 **OS installed, not yet joined to the cluster.** See [Hardware Selection](./01-Hardware-Selection.md) for how this machine was chosen and [Kubernetes Installation](./05-Kubernetes-Installation.md) for the control-plane side this node will join.

Installed on: **Dell OptiPlex 7050 Micro** (second/worker node)

## Hardware recap

- Intel Core i7-6700T (4 cores / 8 threads via Hyper-Threading, 2.8GHz base, up to 3.6GHz boost)
- 16GB DDR4 RAM (included)
- 512GB SSD
- Purchased used/refurbished from seller **iBankonIT, LLC** via eBay, 90-day warranty
- Shipped with Windows 10/11 Pro preinstalled — wiped in favor of Ubuntu Server

Full comparison against the OptiPlex 3050 (passed over) is in [Hardware Selection](./01-Hardware-Selection.md).

## Installation media

Same process as the primary node — see [OS Installation](./02-OS-Installation.md) for the full Rufus/ISO details (Ubuntu Server 26.04 LTS, GPT/UEFI, ISO Image mode). Not repeated here.

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

The same LVM behavior documented on the primary node (see [OS Installation](./02-OS-Installation.md)) showed up again here. Confirmed via `fastfetch`:
```
Disk (/): 6.92 GiB / 97.87 GiB (7%)
```
despite the physical drive being 512GB — the guided installer only assigned ~100GB to the root logical volume and left the rest of the disk as unallocated free space in the volume group. This is the installer's default behavior, not something specific to this hardware — see [LVM Partition Sizing](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-LVM-Partition-Sizing.md) for the full explanation and the standard `lvextend` + `resize2fs` fix.

**Status: ⬜ planned, not yet executed on this node.**

## Next planned steps

1. Apply the LVM root-partition fix (above).
2. Configure a static IP via netplan, following the same process as [Network Configuration](./03-Network-Configuration.md).
3. Install containerd, kernel modules/sysctl settings, and kubelet/kubeadm/kubectl — same prerequisites as [Kubernetes Installation](./05-Kubernetes-Installation.md).
4. Generate a fresh join command on the control-plane node (`kubeadm token create --print-join-command`) and run `kubeadm join` on this node.
5. Once `Ready`, permanently remove the control-plane's `NoSchedule` taint (see [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Kubernetes-Taints-and-Tolerations.md)) so ordinary workloads schedule onto this worker node instead.

## Related

- [Hardware Selection](./01-Hardware-Selection.md) — why the OptiPlex 7050 was chosen over the 3050
- [OS Installation](./02-OS-Installation.md) — full install-media process, shared with the primary node
- [Kubernetes Installation](./05-Kubernetes-Installation.md) — control-plane side this node will join
- [LVM Partition Sizing](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-LVM-Partition-Sizing.md) — companion Guides repository — the root-partition issue and its fix
- [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository — why the control-plane taint matters once this node joins
