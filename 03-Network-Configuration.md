---
tags: [homelab, note, project, networking]
---

# Network Configuration

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

Ubuntu Server's installer (subiquity) writes the network config to a file whose exact name is **not guaranteed** — it's commonly `50-cloud-init.yaml`, but on this install it turned out to be `00-installer-config.yaml`. Guessing the filename wasted a troubleshooting cycle; the reliable way to find it:

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

This is the actual root cause of the earlier "IP resets after every reboot / needs manual `dhcpcd`" symptom: this file only *matched and renamed* the interface by MAC address — it never specified `dhcp4` or a static address at all. It wasn't a case of something silently reverting the config (e.g. cloud-init); the static config from an earlier session had simply never been written into the file the system actually reads on boot.

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
          - 8.8.8.8
          - 1.1.1.1
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

### Troubleshooting note: subnet mismatch

A separate issue surfaced during this process and is worth recording, since it produced the same *symptom* (SSH connection timing out / `ping` returning "Destination host unreachable") but had a completely different cause: the static IP was assigned in the `<SERVER_SUBNET>` range, while the laptop used to connect was on a **different subnet** (`<CLIENT_SUBNET>`, a different gateway). Two devices on different subnets cannot reach each other directly regardless of how correctly the server's own static IP is configured.

**Diagnosis:** compare the server's actual gateway/subnet against the client's:
- On the server: `ip route` (shows the real default gateway in use)
- On a Windows client: `ipconfig` (shows the client's IPv4 address, subnet mask, and default gateway)

If the two devices report different subnets/gateways, the static IP must be re-issued to match the subnet the server is physically connected to — not assumed from an earlier session.

## Result

- Server hostname: `<HOSTNAME>`
- Static IP: `<SERVER_IP>` (confirm this still matches the subnet the server is physically connected to — see troubleshooting note above)
- SSH access: `ssh <USERNAME>@<SERVER_IP>`

## Related

- [SSH Remote Access](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-SSH-Remote-Access.md) — companion Guides repository — how the SSH connection itself works
- [Kubernetes Installation](./05-Kubernetes-Installation.md) — this static IP is the address used for `kubeadm init` and later `kubeadm join`
