---
tags: [homelab, project, verification, kubernetes]
---

# Cluster Verification

> Status: 🟢 **Both nodes verified against the docs.** Two discrepancies found and corrected: the LVM fix had only been applied on the worker, and the DNS fix was documented as DNS-over-TLS when the running config is router DNS.

Live state of both nodes checked against these docs.

## What was checked

| Where | Check |
|---|---|
| Control plane | `kubectl get nodes -o wide` |
| Worker | `/etc/cni/net.d/` contents |
| Worker | absence of `$HOME/.kube` |
| Worker | `iptables -L -n` |
| Both | `df -h`, `vgs`, `lvs` |
| Worker | netplan config in `/etc/netplan/` |
| Both | swap status |
| Both | `containerd` and `kubelet` systemd status |
| Both | kubelet journal |
| Worker | `resolvectl status` and `/etc/systemd/resolved.conf` |

## What was confirmed correct

- **Nodes:** both `Ready`, matching Kubernetes versions.
- **Worker CNI:** `/etc/cni/net.d/` contains only `10-flannel.conflist` — no residue from the accidental `kubeadm init` ([Second Node Setup](./SRV-06-Second-Node-Setup.md)).
- **Worker kubeconfig:** `$HOME/.kube` absent.
- **Worker `iptables`:** no residue from the discarded cluster.
- **Runtime:** `containerd` and `kubelet` active, 3+ days uptime.
- **Kubelet journal:** startup-time garbage-collection errors only; none ongoing.
- **Swap:** disabled on both nodes.

## What didn't match the docs

### 1. LVM was fixed on the worker, but not on the control plane

Status pages implied both nodes were fixed; only the worker was.

- **Worker:** `df -h /` 466G available; `vgs` `VFree 0`.
- **Control plane, before:** installer default — ~100GB root LV of ~473GB in the VG ([OS Installation](./SRV-02-OS-Installation.md)).
- **Control plane, fix** ([LVM Partition Sizing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-LVM-Partition-Sizing.md)):

  ```
  sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
  ```

- **Control plane, after:** root LV spans the full VG.

Updated: [SRV-00](./SRV-00-Project-Overview.md), [SRV-02](./SRV-02-OS-Installation.md), [SRV-06](./SRV-06-Second-Node-Setup.md), root README.

### 2. DNS-over-TLS documented as applied, never configured

On the worker, `resolvectl status` showed `-DNSOverTLS`; `/etc/systemd/resolved.conf` was default. The running fix on both nodes is router DNS (`<GATEWAY_IP>` in netplan `nameservers`). [Network Configuration](./SRV-03-Network-Configuration.md) and [Internet Uplink](../network/NET-02-Internet-Uplink.md) updated; DoT retained as an unapplied alternative.

### 3. Worker documented as DHCP, actually static

Worker netplan: `dhcp4: no`, fixed address. [Second Node Setup](./SRV-06-Second-Node-Setup.md) updated.

## Related

- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the worker join, LVM fix and static-IP follow-up
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the control-plane side of the cluster
- [Network Configuration](./SRV-03-Network-Configuration.md) — the DNS fix, corrected
- [LVM Partition Sizing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-LVM-Partition-Sizing.md) — companion Guides repository
- [DNS-over-TLS and ISP DNS Blocking](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-DNS-over-TLS-and-ISP-DNS-Blocking.md) — companion Guides repository
