---
tags: [homelab, project, second-node, troubleshooting]
---

# Troubleshooting — Second Node Setup

> Companion to [Second Node Setup — Dell OptiPlex 7050 (Worker)](./SRV-06-Second-Node-Setup.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Relates to |
|---|---|
| [No IPv4 address despite a healthy link](#no-ipv4-address-despite-a-healthy-link) | [Network configuration](./SRV-06-Second-Node-Setup.md#network-configuration) |
| [`kubeadm init` run instead of `kubeadm join`](#kubeadm-init-run-instead-of-kubeadm-join) | [Joining the cluster](./SRV-06-Second-Node-Setup.md#joining-the-cluster) |

## No IPv4 address despite a healthy link

**Symptom:** `ip a` showed `enp0s31f6` `UP` with `LOWER_UP` and an IPv6 link-local address, but no IPv4 address.

**Two separate causes, found in sequence:**

1. **Switch port PVID mismatch** — Port 5 VLAN membership correct, PVID still `1` (same class of bug as in [NET-03-TRBL](../network/NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md#vlan-membership-set-pvid-left-at-default)). Fixed: `802.1Q PVID Setting` → `Port 5 → PVID 20`.

2. **No `dhcp4` directive in netplan.** Installer-generated config:

   ```
   sudo cat /etc/netplan/*.yaml
   ```

   ```yaml
   network:
     ethernets:
       enp0s31f6:
         match:
           macaddress: <WORKER_MAC>
         set-name: enp0s31f6
     version: 2
   ```

   Same root cause as on the primary node ([SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md#ip-address-lost-after-every-reboot)): `match`/`set-name` only, no addressing. **Fix:**

   ```
   sudo nano /etc/netplan/00-installer-config.yaml
   ```

   ```yaml
   network:
     ethernets:
       enp0s31f6:
         match:
           macaddress: <WORKER_MAC>
         set-name: enp0s31f6
         dhcp4: true
     version: 2
   ```

   ```
   sudo netplan apply
   ```

   IPv4 address obtained immediately.

**Follow-up:** the DHCP config was later superseded by the static config in [Network configuration](./SRV-06-Second-Node-Setup.md#network-configuration).

## `kubeadm init` run instead of `kubeadm join`

**What happened:** ran on this node:

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

It completed successfully and printed its own `kubeadm join` line, making the error non-obvious. See [Guide: kubeadm init vs. kubeadm join](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubeadm-Init-vs-Join.md).

**Consequence:** the node became the control plane of a separate cluster (own CA, etcd, API server on `<WORKER_IP>:6443`), disconnected from the real one on `<SERVER_IP>`.

**Fix — reset the node:**

```
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
```

- `/etc/cni/net.d` — stale Flannel config from the accidental `init`
- `$HOME/.kube` — kubeconfig for the discarded cluster
- `iptables` — `kube-proxy` chains from the discarded cluster

The node was then joined with a fresh join command — see [Joining the cluster](./SRV-06-Second-Node-Setup.md#joining-the-cluster).

## Related

- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the working procedure
- [NET-03-TRBL](../network/NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md) — the PVID bug on the switch side
- [Guide: kubeadm init vs. kubeadm join](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubeadm-Init-vs-Join.md) — companion Guides repository
