---
tags: [homelab, project, overview]
---

# Server — Project Overview

## Goal

Build a home lab server to develop and demonstrate practical DevOps / infrastructure skills for career purposes (portfolio for recruiters), while also hosting genuinely useful personal services.

## Target workloads

- Personal website / static site hosting
- Cloud storage (Nextcloud-style)
- Minecraft server (small friend group)
- Media streaming (Plex/Jellyfin, direct play primarily)
- A self-hosted RAG (Retrieval-Augmented Generation) stack for experimentation — not a hard requirement, deprioritized after initial hardware planning

## Key architectural decisions

| Decision | Choice | Rationale |
|---|---|---|
| Orchestration | Kubernetes (kubeadm), not k3s | Wanted the closer-to-production experience despite the extra setup complexity. See [Kubernetes vs k3s](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-vs-k3s.md) |
| Container runtime | containerd (not Docker Engine directly) | Kubernetes dropped native Docker support (dockershim) since v1.24. See [Docker vs Containerd](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Docker-vs-Containerd.md) |
| OS | Ubuntu Server 26.04 LTS | Long-term support, huge community, no GUI overhead |
| Storage strategy | No dedicated NAS for now | Budget prioritized toward stronger compute (GMKtec M8); a NAS (e.g. Ugreen NASync) was evaluated but deferred. See [NAS vs Server](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-NAS-vs-Server.md) |

## Hardware summary

See [Hardware Selection](./SRV-01-Hardware-Selection.md) for the full comparison process.

- **Primary node (control plane):** GMKtec M8 — AMD Ryzen 7 PRO 6650H, 16GB LPDDR5, 512GB SSD, dual 2.5GbE
- **Secondary node (worker):** Dell OptiPlex 7050 Micro — Intel i7-6700T (4C/8T), 16GB DDR4, 512GB SSD

## Status at time of writing

- ✅ Hardware purchased — GMKtec M8 (control plane) and Dell OptiPlex 7050 Micro (worker)
- ✅ Ubuntu Server 26.04 LTS installed on GMKtec M8
- ✅ Static IP configured via netplan (subnet mismatch diagnosed and corrected)
- ✅ SSH remote access working
- ✅ Docker installed (for local image builds/testing)
- ✅ containerd + kubeadm + kubelet + kubectl installed
- ✅ Control plane initialized (`kubeadm init`), node `Ready`, Flannel CNI healthy
- ✅ First test workload (nginx) deployed, verified reachable over the network, and cleaned up
- ✅ Second node (`<WORKER_HOSTNAME>`, Dell OptiPlex 7050 Micro) joined the cluster via `kubeadm join` — see [Second Node Setup](./SRV-06-Second-Node-Setup.md) for the full process, including a real mistake made and recovered from along the way
- ✅ Cluster is now two nodes, both `Ready`: `<HOSTNAME>` (control-plane) and `<WORKER_HOSTNAME>` (worker)
- ✅ LVM root-partition under-allocation fixed on both nodes (worker resolved first, control-plane resolved in a later verification pass) — both now use the full disk
- ✅ Helm installed; `kube-prometheus-stack` deployed and collecting metrics from both nodes — see [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md)
- ✅ ingress-nginx + MetalLB give the cluster a real LAN-reachable entry point (`<INGRESS_IP>`); Grafana reachable through it, replacing the earlier `port-forward`-only access
- ✅ Ansible introduced for node provisioning — idempotency verified via a learning example, then a trimmed, production-safe `site.yml` applied to both real cluster nodes with a clean dry-run and real run — see [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md)
- ⬜ No real application workloads deployed yet (site, Nextcloud, Minecraft, etc.)

Continue at [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md) for the current state of the platform, or continue with real workload deployment.
