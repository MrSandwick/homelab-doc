# Homelab Build Log

A chronological, technical log of building a home Kubernetes lab from scratch: hardware selection, OS install, networking, container runtime, and cluster bootstrap via `kubeadm`.

This repo documents *what was done and why* — decisions made, exact commands run, and problems encountered along the way (including real troubleshooting sessions, not just the happy path).

## Contents

1. See [Project Overview](./00-Project-Overview.md)
2. See [Hardware Selection](./01-Hardware-Selection.md)
3. See [OS Installation](./02-OS-Installation.md)
4. See [Network Configuration](./03-Network-Configuration.md)
5. See [Docker Installation](./04-Docker-Installation.md)
6. See [Kubernetes Installation](./05-Kubernetes-Installation.md)

## Stack

- **Hardware:** GMKtec M8 (Ryzen 7 PRO 6650H, 16GB LPDDR5) as control-plane node; Dell OptiPlex 3050 Micro (i5-7500T) planned as worker node
- **OS:** Ubuntu Server 26.04 LTS
- **Container runtime:** containerd (CRI-compliant, used directly by Kubernetes)
- **Orchestration:** Kubernetes via `kubeadm` (not a lightweight distribution — chosen deliberately for closer-to-production setup experience)

## A note on placeholders

Network details (IP addresses, MAC addresses, hostname) are replaced with placeholder tokens like `<SERVER_IP>` throughout these docs. See `.env.example` for the full list of placeholders used. Real values are kept locally in a gitignored `.env` file and are never committed.

## Companion repository

Conceptual explanations of the technologies used here (Kubernetes vs k3s, Docker vs containerd, Linux fundamentals, network security, etc.) live in a separate, access-controlled repository: [homelab-guides](https://github.com/MrSandwick/homelab-guides). Individual entries above link out to the relevant guide where useful. That repo is shared with trusted collaborators rather than being fully public.

## Status

🟡 In progress — control plane not yet initialized. See [Kubernetes Installation](./05-Kubernetes-Installation.md) for the exact point where work currently stands.
