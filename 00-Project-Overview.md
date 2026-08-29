---
tags: [homelab, project, overview]
---

# Project Overview

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
| Orchestration | Kubernetes (kubeadm), not k3s | Wanted the closer-to-production experience despite the extra setup complexity. See [Kubernetes vs k3s](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Kubernetes-vs-k3s.md) |
| Container runtime | containerd (not Docker Engine directly) | Kubernetes dropped native Docker support (dockershim) since v1.24. See [Docker vs Containerd](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-Docker-vs-Containerd.md) |
| OS | Ubuntu Server 26.04 LTS | Long-term support, huge community, no GUI overhead |
| Storage strategy | No dedicated NAS for now | Budget prioritized toward stronger compute (GMKtec M8); a NAS (e.g. Ugreen NASync) was evaluated but deferred. See [NAS vs Server](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-NAS-vs-Server.md) |

## Hardware summary

See [Hardware Selection](./01-Hardware-Selection.md) for the full comparison process.

- **Primary node (control plane):** GMKtec M8 — AMD Ryzen 7 PRO 6650H, 16GB LPDDR5, 512GB SSD, dual 2.5GbE
- **Secondary node (planned):** Dell OptiPlex 3050 Micro — Intel i5-7500T, 8GB DDR4 (RAM upgrade optional/deferred)

## Status at time of writing

- ✅ Hardware purchased (GMKtec M8; OptiPlex under consideration)
- ✅ Ubuntu Server 26.04 LTS installed on GMKtec M8
- ✅ Static IP configured via netplan
- ✅ SSH remote access working
- ✅ Docker installed (for local image builds/testing)
- 🟡 containerd + kubeadm + kubelet + kubectl installed, cluster **not yet initialized**
- ⬜ Second node not yet joined
- ⬜ No workloads deployed yet

Continue at [Kubernetes Installation](./05-Kubernetes-Installation.md) for exact next steps.
