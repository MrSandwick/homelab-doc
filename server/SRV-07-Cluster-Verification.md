---
tags: [homelab, project, verification, kubernetes]
---

# Cluster Verification

> Status: 🟢 **Both nodes verified against the docs.** Two discrepancies found and corrected: the LVM fix had only been applied on the worker, and the DNS fix was documented as DNS-over-TLS when the running config is router DNS.

A live check of both nodes, run to compare the actual running state against what these docs claim. Most of it matched. The two places it didn't are recorded below, since they are the reason several other pages in this log were corrected.

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

- **Nodes:** both `<HOSTNAME>` (control-plane) and `<WORKER_HOSTNAME>` (worker) report `Ready`, on matching Kubernetes versions.
- **CNI on the worker:** `/etc/cni/net.d/` contains only `10-flannel.conflist`. Nothing is left over from the accidental `kubeadm init` documented in [Second Node Setup](./SRV-06-Second-Node-Setup.md).
- **No stale kubeconfig:** `$HOME/.kube` does not exist on the worker, as expected — a worker never needs cluster-admin credentials.
- **Clean `iptables`:** no residue on the worker from the throwaway cluster that `kubeadm init` created.
- **Runtime health:** `containerd` and `kubelet` are both active with 3+ days of uptime.
- **Kubelet journal:** the errors visible in the journal are startup-time garbage-collection noise, not ongoing failures.
- **Swap:** disabled on both nodes, as [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) requires.

## What didn't match the docs

### 1. LVM was fixed on the worker, but not on the control plane

The root-partition under-allocation described in [OS Installation](./SRV-02-OS-Installation.md) and [Second Node Setup](./SRV-06-Second-Node-Setup.md) had been resolved on the worker, but the control-plane node was still running with the installer's default layout — even though the status pages implied both nodes were done.

- **Worker:** `df -h /` showed 466G available and `vgs` showed `VFree 0`.
- **Control plane, before:** the root logical volume was still at the installer default, ~100GB out of ~473GB available in the volume group (the figures recorded at install time in [OS Installation](./SRV-02-OS-Installation.md)); the remaining space was unallocated free space in the volume group.
- **Control plane, fix applied:** the standard procedure from [LVM Partition Sizing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-LVM-Partition-Sizing.md):

  ```
  sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
  ```

- **Control plane, after:** the root logical volume now consumes all free space in the volume group, so both nodes use the full disk.

Status pages ([SRV-00](./SRV-00-Project-Overview.md), [SRV-02](./SRV-02-OS-Installation.md), [SRV-06](./SRV-06-Second-Node-Setup.md), and the root README) were updated to match.

### 2. DNS-over-TLS was documented as applied, but was never configured

[Network Configuration](./SRV-03-Network-Configuration.md) presented DNS-over-TLS via `systemd-resolved` as the fix for the ISP blocking public DNS resolvers. On the worker, `resolvectl status` showed `-DNSOverTLS` and `/etc/systemd/resolved.conf` was untouched (default). DoT was investigated but never carried through.

The fix actually running on both nodes is simpler: DNS is pointed at the router's own address (`<GATEWAY_IP>`) through netplan's `nameservers` block. [Network Configuration](./SRV-03-Network-Configuration.md) and [Internet Uplink](../network/NET-02-Internet-Uplink.md) now describe router DNS as the applied fix and keep DoT as a documented alternative.

### Related correction: the worker is on a static IP

[Second Node Setup](./SRV-06-Second-Node-Setup.md) described the worker as left on DHCP. The netplan config on the worker showed `dhcp4: no` with a fixed address, matching the primary node. The page now says so and keeps the earlier `dhcp4: true` troubleshooting as historical record.

## Related

- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the worker join, LVM fix and static-IP follow-up
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the control-plane side of the cluster
- [Network Configuration](./SRV-03-Network-Configuration.md) — the DNS fix, corrected
- [LVM Partition Sizing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-LVM-Partition-Sizing.md) — companion Guides repository
- [DNS-over-TLS and ISP DNS Blocking](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-DNS-over-TLS-and-ISP-DNS-Blocking.md) — companion Guides repository
