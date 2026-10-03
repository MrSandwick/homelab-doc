---
tags: [homelab, project, network, troubleshooting]
---

# Troubleshooting — Server Network Configuration

> Companion to [Server — Network Configuration](../server/SRV-03-Network-Configuration.md), which records only the working configuration. This file records the problems hit while getting there, in the order they occurred.

## Contents

| Incident | Relates to |
|---|---|
| [IP address lost after every reboot](#ip-address-lost-after-every-reboot) | [Making the IP static](../server/SRV-03-Network-Configuration.md#making-the-ip-static) |
| [Subnet mismatch between server and client](#subnet-mismatch-between-server-and-client) | [Making the IP static](../server/SRV-03-Network-Configuration.md#making-the-ip-static) |
| [ISP blocking public DNS resolvers](#isp-blocking-public-dns-resolvers) | [DNS](../server/SRV-03-Network-Configuration.md#dns) |

## IP address lost after every reboot

**Symptom:** the address reset after every reboot and had to be re-acquired manually with `dhclient` / `dhcpcd`.

**Diagnosis:** the netplan filename was guessed (`50-cloud-init.yaml`), which cost a troubleshooting cycle; on this install the file is `00-installer-config.yaml`.

```
ls /etc/netplan/
sudo cat /etc/netplan/*.yaml
```

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

**Root cause:** the installer-written file only matched and renamed the interface by MAC address — it specified neither `dhcp4` nor a static address. Nothing was reverting the config (e.g. cloud-init); the static config from an earlier session had never been written into the file the system reads on boot.

**Fix:** addressing directives added inside the existing `eno1` entry of that file — see [Making the IP static](../server/SRV-03-Network-Configuration.md#making-the-ip-static).

## Subnet mismatch between server and client

**Symptom:** SSH timed out and `ping` returned `Destination host unreachable` — the same symptom as a missing static config, with a different cause.

**Diagnosis:** compare the server's gateway/subnet with the client's.

- On the server: `ip route` (default gateway actually in use)
- On a Windows client: `ipconfig` (IPv4 address, subnet mask, default gateway)

**Root cause:** the static IP had been assigned in the `<SERVER_SUBNET>` range while the client laptop was on a different subnet (`<CLIENT_SUBNET>`, different gateway). Two devices on different subnets cannot reach each other directly, regardless of how correctly the server's own address is configured.

**Fix:** the static IP was re-issued in the subnet the server is physically connected to, not assumed from an earlier session.

## ISP blocking public DNS resolvers

**Symptom:** name resolution stopped working on the server (`ping google.com` → `Temporary failure in name resolution`), noticed around the time the second node was connected — the timing initially suggested a networking regression from that change. At that point netplan `nameservers` pointed at `8.8.8.8` and `1.1.1.1`.

**Diagnosis:**

```
ping 8.8.8.8                        # succeeds — basic connectivity is fine
nslookup google.com 8.8.8.8         # "communications error ... timed out"
curl -v https://8.8.8.8 --insecure  # "Connection refused" in ~36ms
nc -zv -w3 8.8.8.8 53               # Connection refused
nc -zv -w3 1.1.1.1 53               # Connection refused (same result)
ping google.com                     # (from a different device) resolves and replies normally via the device's own DNS
```

`ufw` was inactive, `iptables` rules on this host only targeted internal Kubernetes service IPs, and no Omada ACL was in place. `Connection refused` arriving in ~36ms is an active rejection from something close by, not a lost-packet timeout — and it affected `8.8.8.8` (Google) and `1.1.1.1` (Cloudflare) identically, while an ordinary Google server IP (`142.250.217.110`) pinged normally.

**Root cause:** the ISP is a cellular 5G Home Internet product (see [Internet Uplink](../network/NET-02-Internet-Uplink.md)) that blocks direct connections to well-known public DNS resolvers by IP. Standard DNS (port 53) and a plain TLS connection (port 443) to `8.8.8.8` / `1.1.1.1` were both rejected; only the well-known DNS IPs were targeted, not general traffic to those providers' other services.

**Fix (applied on both nodes):** netplan `nameservers` set to the router, replacing `8.8.8.8` / `1.1.1.1`:

```yaml
      nameservers:
        addresses:
          - <GATEWAY_IP>
```

```
sudo netplan try
sudo netplan apply
```

Confirmed running on both nodes in [Cluster Verification](../server/SRV-07-Cluster-Verification.md).

**Alternative investigated, not applied — DNS-over-TLS via `systemd-resolved`.** Not configured on either node (`resolvectl status`: `-DNSOverTLS`; `resolved.conf` default); the router-DNS fix was sufficient. Reference config:

```
sudo nano /etc/systemd/resolved.conf
```

```ini
[Resolve]
DNS=9.9.9.9#dns.quad9.net 1.1.1.1#cloudflare-dns.com
DNSOverTLS=opportunistic
```

```
sudo systemctl restart systemd-resolved
resolvectl status   # expect "+DNSOverTLS" on the active link
```

`opportunistic` rather than `yes`, to fall back instead of failing if TLS is unavailable. See [Guide-DNS-over-TLS-and-ISP-DNS-Blocking](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-DNS-over-TLS-and-ISP-DNS-Blocking.md).

**Caveat:** host DNS settings do not propagate into pod/container network namespaces; the same blocking inside a pod would need separate DNS configuration.

**Recurrence:** the same blocking later broke the Helm install script — see [SRV-08-TRBL](./SRV-08-TRBL-Helm-Observability-Ingress.md#helm-install-script-failed-to-resolve).

## Related

- [Server — Network Configuration](../server/SRV-03-Network-Configuration.md) — the working configuration
- [Internet Uplink](../network/NET-02-Internet-Uplink.md) — the ISP connection behind the DNS blocking
- [Cluster Verification](../server/SRV-07-Cluster-Verification.md) — confirmed which DNS fix is running
