---
tags: [homelab, project, kubernetes, gpu, hardware]
---

# Node Profile — GPU Laptop Worker

> Status: 🟢 Joined, healthy, GPU verified usable by pods. This doc covers *what this node is and the operational trade-offs of running it*; see [Second Node Setup](./SRV-06-Second-Node-Setup.md) for the join process it followed and [GPU Node Setup](./SRV-12-GPU-Node-Setup.md) for how its GPU was made usable by pods.

> **Standing decision (October 2026):** the laptop is also used and administered by another person, so it is intentionally kept out of the Ansible inventory and out of scheduling. It stays cordoned and receives no workloads or pods beyond its node-level DaemonSets for the foreseeable future. The practices below describe how it could be used; they are not currently in effect. See [ArgoCD and GitOps — Open items](./SRV-13-ArgoCD-GitOps.md#open-items).

## What this node is

Unlike the cluster's other two nodes — a GMKtec M8 and a Dell OptiPlex 7050 Micro, both purchased specifically for this project (see [Hardware Selection](./SRV-01-Hardware-Selection.md)) — this third node is an existing personal laptop, repurposed rather than bought for the cluster. It was added specifically because it has a discrete NVIDIA GPU, which neither of the other two nodes has, making it the only practical path to the self-hosted RAG stack listed in [Project Overview](./SRV-00-Project-Overview.md).

### Hardware

- **Model:** Acer Nitro AN515-57
- **CPU:** Intel Core i5-11400H (6 cores / 12 threads, up to 4.5GHz)
- **RAM:** 14.89 GiB (~15GB) DDR4
- **GPU:** NVIDIA GeForce RTX 3050 Ti Mobile, 4GB VRAM (plus integrated Intel UHD Graphics, unused by the cluster)
- **Disk:** 466GB, ext4
- **OS:** Ubuntu 26.04.1 LTS **Desktop** (GNOME, Wayland) — not Server, unlike the other two nodes
- **Hostname:** `<GPU_HOSTNAME>`

## The core tension: daily-driver laptop vs. always-on cluster node

This is the one node in the cluster that's also actively used for everyday work — browsing, development, personal use — at the same time it's expected to run scheduled pods. That combination introduces trade-offs the other two (headless, single-purpose) nodes don't have.

### The swap decision

At idle, this machine already uses ~2GiB of RAM just for the GNOME desktop session — before any browser tabs, IDE, or scheduled pods are factored in. Kubernetes requires swap disabled (see [Kubernetes Swap Requirement](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Swap-Requirement.md) for why), but on a 15GB machine also running a full desktop environment, removing swap's out-of-memory safety net is a more meaningful trade-off here than on the two headless server nodes, where the only consumers of RAM are predictable background services.

**Decision:** swap was disabled (standard requirement, non-negotiable for `kubeadm`), but two operational practices were adopted specifically to compensate on this node:

1. **`kubectl cordon` during heavy personal use.** Cordoning marks the node unschedulable for *new* pods without affecting already-running ones or removing it from the cluster:

   ```bash
   kubectl cordon <GPU_HOSTNAME>
   # ... do memory-intensive personal work ...
   kubectl uncordon <GPU_HOSTNAME>
   ```

   This is a manual practice, not automated — there's no trigger currently watching for "the owner is doing something memory-heavy" and cordoning automatically.

2. **Explicit memory `resources.limits`/`requests` on any pod scheduled here**, rather than relying on defaults, so a single workload can't silently consume enough RAM to destabilize the desktop session:

   ```yaml
   resources:
     limits:
       memory: "4Gi"
     requests:
       memory: "2Gi"
   ```

Neither practice has been stress-tested under genuinely heavy simultaneous load (e.g. a large GPU workload running while the desktop is also under memory pressure) — this remains a known, accepted risk rather than a fully solved problem.

## Desktop-specific installation notes

A few things differed from the Server-OS installs on the other two nodes, worth knowing if a future node is set up the same way:

- **Swap is a swapfile by default on Ubuntu 26.04 Desktop** (confirmed — not zram-based), so the standard `swapoff -a` + commenting the `/etc/fstab` entry (same method used on the other two nodes, see [Kubernetes Installation, Step 1](./SRV-05-Kubernetes-Installation.md#step-1--disable-swap)) applies unchanged.
- **The NVIDIA driver was already installed and working** before any Kubernetes-related setup began — Ubuntu Desktop handles this automatically for display output, unlike a headless server where it has to be installed deliberately. Confirmed with `nvidia-smi` before starting the GPU-enablement work in [GPU Node Setup](./SRV-12-GPU-Node-Setup.md).
- **This install did not hit the LVM root-partition under-allocation issue** seen on the other two nodes (see [OS Installation](./SRV-02-OS-Installation.md)) — `df -h /` showed the full ~466GB available from the start. The likely explanation is a different disk-partitioning path taken during this particular Desktop install versus the Server installer's guided-LVM flow, though this wasn't confirmed in detail.

## Related

- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the join process this node followed (the same process applies regardless of Desktop vs. Server OS)
- [GPU Node Setup](./SRV-12-GPU-Node-Setup.md) — making the GPU usable by pods
- [Hardware Selection](./SRV-01-Hardware-Selection.md) — the two purchased nodes this one complements
- [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Taints-and-Tolerations.md) — a more automated alternative to manual cordon/uncordon worth evaluating later (e.g. a taint reserving this node for GPU workloads specifically)
