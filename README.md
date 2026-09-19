# Homelab — Documentation

A home lab built end to end and documented as it was built: a **Kubernetes server** (`kubeadm` cluster on two mini-PCs) behind a **segmented, RADIUS-authenticated network** (VLANs, WPA2-Enterprise Wi-Fi, per-VLAN isolation).

The docs record *what was done and why* — decisions and rejected alternatives, the exact commands run, and the real problems hit along the way (not just the happy path).

## How this repository is organised

The project has two layers, each documented in its own folder. Every file name carries its layer prefix, so no two documents in the repository share a name or a number:

| Folder | Prefix | Layer | Start here |
|---|---|---|---|
| [`server/`](server/) | `SRV-` | Hardware, OS, Docker, Kubernetes cluster | [SRV-README](server/SRV-README.md) |
| [`network/`](network/) | `NET-` | ISP uplink, VLANs, router/switch/AP, FreeRADIUS, monitoring | [NET-README](network/NET-README.md) |

Numbers are only meaningful *within* a layer (`SRV-03` is the third server doc, `NET-03` the third network doc) — the prefix is what identifies the document.

## Architecture at a glance

```
Internet
   |
ISP gateway (5G Home Internet, own NAT)          network/
   |
TP-Link ER605 router  (VLANs, DHCP, firewall)    network/
   |
TP-Link TL-SG108E switch  (802.1Q trunk)         network/
   |
   +-- EAP610 access point -- Users SSID  (VLAN 10, WPA-Personal)     network/
   |                      \-- Admin SSID  (VLAN 20, WPA2-Enterprise)  network/
   |
   +-- Admin VLAN 20 -- GMKtec M8 (Kubernetes control plane + FreeRADIUS)   server/ + network/
                     \-- Dell OptiPlex 7050 (Kubernetes worker)             server/
```

```
LAN traffic into the cluster:
<INGRESS_IP> (MetalLB) -> ingress-nginx -> Grafana (path: /)
                                        -> [future services]
```

The FreeRADIUS server that authenticates Wi-Fi clients in `network/` runs on the same physical machine as the Kubernetes control plane in `server/`.

## Documentation

### Server — Kubernetes cluster

| Doc | Covers |
|---|---|
| [SRV-00 Project Overview](server/SRV-00-Project-Overview.md) | Goals, target workloads, key decisions, status |
| [SRV-01 Hardware Selection](server/SRV-01-Hardware-Selection.md) | Options compared, GMKtec M8 + OptiPlex 7050 |
| [SRV-02 OS Installation](server/SRV-02-OS-Installation.md) | Ubuntu Server 26.04 LTS, LVM layout |
| [SRV-03 Network Configuration](server/SRV-03-Network-Configuration.md) | Static IP with netplan, subnet-mismatch troubleshooting |
| [SRV-04 Docker Installation](server/SRV-04-Docker-Installation.md) | Docker for local image builds |
| [SRV-05 Kubernetes Installation](server/SRV-05-Kubernetes-Installation.md) | containerd, `kubeadm init`, Flannel, first workload |
| [SRV-06 Second Node Setup](server/SRV-06-Second-Node-Setup.md) | OptiPlex worker node preparation |
| [SRV-07 Cluster Verification](server/SRV-07-Cluster-Verification.md) | Live-state check against the docs; LVM fixed on the control plane; DNS fix corrected |
| [SRV-08 Helm, Observability, and Ingress](server/SRV-08-Helm-Observability-Ingress.md) | Helm, kube-prometheus-stack, ingress-nginx + MetalLB |

### Network — VLANs and RADIUS

| Doc | Covers |
|---|---|
| [NET-00 Project Overview](network/NET-00-Project-Overview.md) | Goals, requirements, key decisions, status |
| [NET-01 Hardware Selection](network/NET-01-Hardware-Selection.md) | ER605, TL-SG108E, EAP610 |
| [NET-02 Internet Uplink](network/NET-02-Internet-Uplink.md) | Double NAT behind the ISP gateway |
| [NET-03 VLAN Design and Switch Configuration](network/NET-03-VLAN-Design-and-Switch-Configuration.md) | VLAN plan, trunking, PVID troubleshooting |
| [NET-04 Router Configuration](network/NET-04-Router-Configuration.md) | ER605, Omada controller adoption |
| [NET-05 FreeRADIUS Installation](network/NET-05-FreeRADIUS-Installation.md) | MAC-based RADIUS users, EAP troubleshooting |
| [NET-06 Wireless / RADIUS Integration](network/NET-06-Wireless-RADIUS-Integration.md) | Admin and Users SSIDs |
| [NET-07 Access Control and Isolation](network/NET-07-Access-Control-and-Isolation.md) | Network isolation, planned ACLs |
| [NET-08 Traffic Monitoring](network/NET-08-Monitoring.md) | Port mirroring, ntopng |

## Stack

- **Server:** GMKtec M8 (Ryzen 7 PRO 6650H, 16 GB) control plane; Dell OptiPlex 7050 Micro (i7-6700T, 16 GB) worker; Ubuntu Server 26.04 LTS; containerd; Kubernetes via `kubeadm` with Flannel; Helm; `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager, node-exporter); ingress-nginx + MetalLB
- **Network:** 5G Home Internet gateway (double NAT); TP-Link Omada ER605 router, TL-SG108E switch, EAP610 Wi-Fi 6 access point; FreeRADIUS on the server

## Status

| Layer | Working | In progress / planned |
|---|---|---|
| Server | Two-node cluster, both `Ready` (control plane + worker), Flannel CNI healthy, workload scheduling verified on the worker, Helm, `kube-prometheus-stack` monitoring both nodes, Grafana reachable on the LAN via ingress-nginx + MetalLB | Real application workloads (site, Nextcloud, Minecraft, etc.) |
| Network | Double-NAT uplink, VLANs 10/20, Users/Admin isolation, management ACLs, ntopng traffic monitoring — all verified | WPA2-Enterprise SSID to be recreated after an AP factory reset; login-free MAC-based auth; IDS/IPS |

Each layer's overview ([SRV-00](server/SRV-00-Project-Overview.md), [NET-00](network/NET-00-Project-Overview.md)) has the full status list.

## A note on placeholders

IP addresses, MAC addresses, hostnames, secrets, and Wi-Fi passwords are replaced with placeholder tokens like `<SERVER_IP>` throughout these docs. See [`.env.example`](.env.example) for the full list. Real values are kept locally in a gitignored `.env` file and are never committed.

## License

[MIT](LICENSE)
