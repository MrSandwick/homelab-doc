# Server & Kubernetes Build Log

A chronological, technical log of building a home Kubernetes lab from scratch: hardware selection, OS install, networking, container runtime, and cluster bootstrap via `kubeadm`.

This section documents *what was done and why* — decisions made, exact commands run, and problems encountered along the way.

Each numbered doc records the working procedure. The problems hit along the way — symptom, root cause, fix — are kept in a companion file with the same number and a `TRBL` tag in the [`troubleshooting/`](../troubleshooting/) folder (`troubleshooting/SRV-13-TRBL-ArgoCD-GitOps.md` for `SRV-13-ArgoCD-GitOps.md`), linked from the doc it belongs to.

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
10. See [Ansible Node Provisioning](./SRV-09-Ansible-Node-Provisioning.md)
11. See [First Real Workload](./SRV-10-First-Real-Workload.md)
12. See [Node Profile — GPU Laptop Worker](./SRV-11-GPU-Laptop-Node-Profile.md)
13. See [GPU Node Setup](./SRV-12-GPU-Node-Setup.md)
14. See [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md)

Troubleshooting files:

- [SRV-03-TRBL](../troubleshooting/SRV-03-TRBL-Network-Configuration.md) — IP lost on reboot, subnet mismatch, ISP blocking public DNS resolvers
- [SRV-05-TRBL](../troubleshooting/SRV-05-TRBL-Kubernetes-Installation.md) — `conntrack` preflight, Flannel crash loop, pod `Pending` on the taint, `port-forward`
- [SRV-06-TRBL](../troubleshooting/SRV-06-TRBL-Second-Node-Setup.md) — no IPv4 address, `kubeadm init` run instead of `join`
- [SRV-08-TRBL](../troubleshooting/SRV-08-TRBL-Helm-Observability-Ingress.md) — Helm script DNS failure, interrupted `helm install`
- [SRV-09-TRBL](../troubleshooting/SRV-09-TRBL-Ansible-Node-Provisioning.md) — SSH host-key and auth failures, `sudo-rs`
- [SRV-10-TRBL](../troubleshooting/SRV-10-TRBL-First-Real-Workload.md) — first deployment pinned to the control plane
- [SRV-12-TRBL](../troubleshooting/SRV-12-TRBL-GPU-Node-Setup.md) — empty `nvidia-ctk` config, Device Plugin crash loops
- [SRV-13-TRBL](../troubleshooting/SRV-13-TRBL-ArgoCD-GitOps.md) — pods on an unexpected node, GitHub push failures

## Stack

- **Hardware:** GMKtec M8 (Ryzen 7 PRO 6650H, 16GB LPDDR5) as control-plane node; Dell OptiPlex 7050 Micro (i7-6700T, 16GB DDR4) and an Acer Nitro AN515-57 laptop (i5-11400H, RTX 3050 Ti Mobile) as worker nodes
- **OS:** Ubuntu Server 26.04 LTS on the two purchased nodes; Ubuntu 26.04 Desktop on the laptop worker
- **GPU:** NVIDIA Container Toolkit + Device Plugin on the laptop worker, exposing `nvidia.com/gpu` to pods
- **Container runtime:** containerd (CRI-compliant, used directly by Kubernetes)
- **Orchestration:** Kubernetes via `kubeadm` (not a lightweight distribution — chosen deliberately for closer-to-production setup experience)
- **Package management:** Helm
- **Observability:** `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics)
- **Ingress / bare-metal LoadBalancer:** ingress-nginx + MetalLB
- **Configuration management:** Ansible (control node on the primary node; provisions base packages, kernel modules/sysctl, and Kubernetes package installation — deliberately scoped to exclude DNS and containerd-config management to avoid conflicting with the live, manually-verified configuration in those areas)
- **GitOps:** ArgoCD, syncing from a public `homelab-gitops` repository

Layer-by-layer overview of the stack: [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md) (companion Guides repository).

## A note on placeholders

Network details (IP addresses, MAC addresses, hostname) are replaced with placeholder tokens like `<SERVER_IP>` throughout these docs. See `.env.example` for the full list of placeholders used. Real values are kept locally in a gitignored `.env` file and are never committed.

## Companion repository

Conceptual explanations of the technologies used here (Kubernetes vs k3s, Docker vs containerd, Linux fundamentals, network security, etc.) live in a separate, access-controlled repository: [OVault → homelab-docs/homelab-guides](https://github.com/MrSandwick/OVault/tree/main/homelab-docs/homelab-guides), under `server/`. Individual entries above link out to the relevant guide where useful. That repo is shared with trusted collaborators rather than being fully public.

## Status

🟢 The cluster is now three nodes, all `Ready` (`kubeadm`, containerd, Flannel CNI healthy on each): the GMKtec M8 control plane, the Dell OptiPlex 7050 Micro worker, and a repurposed Acer Nitro laptop worker. A test workload was deployed on a worker node and verified reachable. See [Second Node Setup](./SRV-06-Second-Node-Setup.md) for the full join process ([SRV-06-TRBL](../troubleshooting/SRV-06-TRBL-Second-Node-Setup.md) for the `kubeadm init`-vs-`join` mistake and recovery). The laptop's GPU (NVIDIA RTX 3050 Ti Mobile) is fully usable by pods — see [GPU Node Setup](./SRV-12-GPU-Node-Setup.md).
