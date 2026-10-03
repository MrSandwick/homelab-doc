---
tags: [homelab, project, network]
---

# Server — Network Configuration

## Contents

1. [Initial state](#initial-state)
2. [Interfaces detected](#interfaces-detected)
3. [DHCP lease (initial)](#dhcp-lease-initial)
4. [Making the IP static](#making-the-ip-static)
5. [Result](#result)
6. [DNS](#dns)
7. [Related](#related)

Problems hit during this work are recorded separately in [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md).

## Initial state

The installer was completed with **"Continue without network"** — no Ethernet cable was connected during OS install. Network was configured post-install.

## Interfaces detected

GMKtec M8 exposes three network interfaces (confirmed via `ip a`):

| Interface | Type | Notes |
|---|---|---|
| `eno1` | Ethernet (2.5GbE, Realtek RTL8125) | Used — connected to router |
| `enp2s0` | Ethernet (2.5GbE, Realtek RTL8125) | Second NIC, unused for now |
| `wlp4s0` | WiFi (MediaTek MT7922) | Unused — wired preferred for a server |

## DHCP lease (initial)

After connecting the cable and manually triggering a lease:
```
sudo dhclient eno1
```
The interface received `<SERVER_IP>` from the router (`<GATEWAY_IP>`).

## Making the IP static

Static IP was configured via **netplan** (Ubuntu's default network configuration tool) rather than a router-side DHCP reservation, to keep the configuration self-contained on the server.

### Finding the correct config file

Ubuntu Server's installer (subiquity) writes the network config to a file whose exact name is **not guaranteed** — it's commonly `50-cloud-init.yaml`, but on this install it is `00-installer-config.yaml`. To find it:

```
ls /etc/netplan/
```

Then inspect what's actually inside it (root-owned, needs `sudo` to read):
```
sudo cat /etc/netplan/*.yaml
```

On this server, the installer had written:
```yaml
network:
  ethernets:
    eno1:
      match:
        macaddress: <SERVER_MAC>
      set-name: eno1
    enp2s0:
      accept-ra: true
  version: 2
  wifis: {}
```

The file only matches and renames the interface by MAC address — it specifies neither `dhcp4` nor a static address, so nothing persists across a reboot. See [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md#ip-address-lost-after-every-reboot).

### Correct config

Edit the file found above (**do not** delete the existing `match`/`set-name` block — keep it, and add the addressing directives inside the same `eno1` entry):

```
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  ethernets:
    eno1:
      match:
        macaddress: <SERVER_MAC>
      set-name: eno1
      dhcp4: no
      addresses:
        - <SERVER_IP>/24
      routes:
        - to: default
          via: <GATEWAY_IP>
      nameservers:
        addresses:
          - <GATEWAY_IP>
    enp2s0:
      accept-ra: true
  version: 2
  wifis: {}
```

Applied safely with a rollback window:
```
sudo netplan try
sudo netplan apply
```

`netplan try` is important — it auto-reverts after ~20 seconds if not confirmed, protecting against a misconfiguration that would cut off SSH access.

### Verifying it survives a reboot

```
sudo reboot
```
then, after the server has had time to boot:
```
ssh <USERNAME>@<SERVER_IP>
```
If this connects without needing a manual `sudo dhclient eno1` / `sudo dhcpcd eno1` first, the static config is correctly persisted.

The static IP must be in the subnet the server is physically connected to; an address issued in the wrong subnet was diagnosed and corrected — see [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md#subnet-mismatch-between-server-and-client).

## Result

- Server hostname: `<HOSTNAME>`
- Static IP: `<SERVER_IP>` — re-issued in the correct subnet after a subnet mismatch was diagnosed ([SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md#subnet-mismatch-between-server-and-client)); confirmed reachable via `ping` and `ssh` from a same-subnet client, and confirmed persistent across reboot during the `kubeadm init` process
- SSH access: `ssh <USERNAME>@<SERVER_IP>`

## DNS

netplan `nameservers` points at the router (`<GATEWAY_IP>`) on both nodes, not at public resolvers: the ISP blocks direct connections to `8.8.8.8` / `1.1.1.1`. The config originally used those two addresses and was changed after resolution failed — see [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md#isp-blocking-public-dns-resolvers), which also records the DNS-over-TLS alternative that was investigated and not applied.

Confirmed running on both nodes in [Cluster Verification](./SRV-07-Cluster-Verification.md).

## Related

- [SSH Remote Access](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-SSH-Remote-Access.md) — companion Guides repository — how the SSH connection itself works
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — this static IP is the address used for `kubeadm init` and later `kubeadm join`
- [Guide-DNS-over-TLS-and-ISP-DNS-Blocking](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-DNS-over-TLS-and-ISP-DNS-Blocking.md) — companion Guides repository
- [Cluster Verification](./SRV-07-Cluster-Verification.md) — confirmed which DNS fix is running
- [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md) — troubleshooting for this doc
