---
tags: [homelab-project, homelab, note, project, hardware]
---

# Hardware Selection

## Constraints

- Apartment living → noise and heat ruled out rack-mount enterprise servers (loud fans, high idle power draw — a purely practical constraint, not covered in a separate guide).
- Initial budget target: $400–500, later revised upward for a stronger single node.
- Primary goal shifted over the course of planning from "RAG-focused" to "general-purpose services with occasional light AI experimentation."

## Options evaluated (chronological)

| Model | CPU | RAM | Verdict |
|---|---|---|---|
| KAMRUI AK1 Plus | Celeron N5095 | 16GB (soldered) | Rejected — outdated CPU, no 2.5GbE |
| Beelink SER3 | Ryzen 3 3200U | 8GB | Rejected — too weak (2C/4T) |
| Beelink Mini S12 | Intel N95 | 8GB | Rejected — weak CPU, RAM ceiling too low |
| Beelink EQ (N150) | Intel N150 | 12GB | Rejected — no standout advantage |
| Beelink SER5 (5560U / 5500U) | Ryzen 5, 6C/12T | 16GB (expandable to 64GB) | Strong candidate, ultimately passed on price (~$390 for one node) |
| GMKtec G10 | Ryzen 5 3500U | 16GB | Viable budget second node |
| BOSGAME E5 | Ryzen 3 5300U | 16GB | Viable, notable for dual LAN |
| GMKtec G11 | Ryzen Embedded R2514 | 16GB | Good dual 2.5GbE, embedded-grade stability |
| **GMKtec M8** | **Ryzen 7 PRO 6650H, 6C/12T** | **16GB LPDDR5** | **✅ Purchased** — best CPU/network combo seen, ~$440 |
| Dell OptiPlex 3050 Micro (used) | i5-7500T | 8GB DDR4 | **Planned second node**, ~$138.50 |

## Final decision

**Primary node — GMKtec M8**
- AMD Ryzen 7 PRO 6650H (6 cores / 12 threads, boost to 4.5GHz)
- 16GB LPDDR5 RAM (soldered — not user-upgradeable)
- 512GB PCIe SSD
- Dual 2.5GbE NIC
- Radeon 660M integrated graphics
- Oculink port (future eGPU expansion path)
- USB4

**Secondary node — Dell OptiPlex 3050 Micro (used, $138.50)**
- Intel Core i5-7500T (4C/4T, 2.7GHz)
- 8GB DDR4 (RAM upgrade to 16GB considered, not yet purchased)
- 256GB SSD
- Standard Gigabit Ethernet (not 2.5GbE — will be the network bottleneck between nodes)

## Rejected form factors

- **1U/2U rack servers** — too loud for apartment use (40-60+ dB fans), high idle power draw (150-300W vs 10-20W for mini PCs).
- **Thin clients** (e.g. Dell OptiPlex 3000 Thin Client) — explicitly avoided; these ship with tiny storage (32GB) and are designed as terminals for VDI, not as standalone compute nodes.
- **Dedicated NAS (Ugreen NASync DH2300)** — evaluated but rejected for this budget round. It's storage-only (ARM CPU, 4GB fixed RAM, no meaningful compute capability) and would have consumed budget without adding compute power. See [NAS vs Server](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-NAS-vs-Server.md).

## Networking note

The two nodes have asymmetric network capability: GMKtec M8 has dual 2.5GbE, OptiPlex has standard Gigabit. Cluster inter-node traffic will be capped by the slower link. Acceptable for a home-scale cluster with light-to-moderate workloads.
