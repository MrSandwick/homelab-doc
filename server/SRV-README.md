# Server & Kubernetes Build Log

A chronological, technical log of building a home Kubernetes lab from scratch: hardware selection, OS install, networking, container runtime, and cluster bootstrap via `kubeadm`.

This section documents *what was done and why* — decisions made, exact commands run, and problems encountered along the way (including real troubleshooting sessions, not just the happy path).

## Contents

1. See [Project Overview](./SRV-00-Project-Overview.md)
2. See [Hardware Selection](./SRV-01-Hardware-Selection.md)
3. See [OS Installation](./SRV-02-OS-Installation.md)
4. See [Network Configuration](./SRV-03-Network-Configuration.md)
5. See [Docker Installation](./SRV-04-Docker-Installation.md)
6. See [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md)
7. See [Second Node Setup](./SRV-06-Second-Node-Setup.md)
8. See [Cluster Verification](./SRV-07-Cluster-Verification.md)
9. See [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md)

## Stack

- **Hardware:** GMKtec M8 (Ryzen 7 PRO 6650H, 16GB LPDDR5) as control-plane node; Dell OptiPlex 7050 Micro (i7-6700T, 16GB DDR4) as worker node
- **OS:** Ubuntu Server 26.04 LTS
- **Container runtime:** containerd (CRI-compliant, used directly by Kubernetes)
- **Orchestration:** Kubernetes via `kubeadm` (not a lightweight distribution — chosen deliberately for closer-to-production setup experience)
- **Package management:** Helm
- **Observability:** `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics)
- **Ingress / bare-metal LoadBalancer:** ingress-nginx + MetalLB

## A note on placeholders

Network details (IP addresses, MAC addresses, hostname) are replaced with placeholder tokens like `<SERVER_IP>` throughout these docs. See `.env.example` for the full list of placeholders used. Real values are kept locally in a gitignored `.env` file and are never committed.

## Companion repository

Conceptual explanations of the technologies used here (Kubernetes vs k3s, Docker vs containerd, Linux fundamentals, network security, etc.) live in a separate, access-controlled repository: [OVault → homelab-docs/homelab-guides](https://github.com/MrSandwick/OVault/tree/main/homelab-docs/homelab-guides), under `server/`. Individual entries above link out to the relevant guide where useful. That repo is shared with trusted collaborators rather than being fully public.

## Status

🟢 The cluster is now two nodes, both `Ready`: the GMKtec M8 control plane and the Dell OptiPlex 7050 Micro worker (`kubeadm`, containerd, Flannel CNI all healthy on both). A test workload was deployed on the worker node and verified reachable. See [Second Node Setup](./SRV-06-Second-Node-Setup.md) for the full join process, including a real `kubeadm init`-vs-`join` mistake and recovery.
