---
tags: [homelab-project, homelab, note, project, networking, vlan, troubleshooting]
---

# Troubleshooting — VLAN Design and Switch Configuration

> Companion to [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md), which records only the working configuration. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Relates to |
|---|---|
| [VLAN membership set, PVID left at default](#vlan-membership-set-pvid-left-at-default) | [Switch-side configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md#switch-side-configuration-tl-sg108e-via-easy-smart-utility) |
| [Switch management IP confusion](#switch-management-ip-confusion) | [Switch-side configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md#switch-side-configuration-tl-sg108e-via-easy-smart-utility) |
| [Same PVID bug recurred when adding a second server](#same-pvid-bug-recurred-when-adding-a-second-server) | [Switch-side configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md#switch-side-configuration-tl-sg108e-via-easy-smart-utility) |

## VLAN membership set, PVID left at default

**Symptom:** a laptop plugged into the port configured for VLAN 20 (server/admin), with correct Untagged membership already set, still received a DHCP lease from VLAN 1 (`<MGMT_SUBNET>.100`, confirmed via `ipconfig`) instead of VLAN 20.

**Root cause:** the port's PVID was still at its default value of `1`. Setting a port's VLAN *membership* to Untagged is not sufficient by itself — the port's **PVID** (Port VLAN ID) must be set to the same VLAN, because PVID determines which VLAN untagged *incoming* traffic from the connected device is placed into. The membership table alone controls how frames are handled going the other direction.

**Fix:** `VLAN → 802.1Q PVID Setting`, set the port's PVID to `20`, save, then force the client to re-request an address:

```
ipconfig /release
ipconfig /renew
```

The client immediately obtained a `<ADMIN_SUBNET>.x` address. The same PVID check was subsequently applied to every access port on the switch.

## Switch management IP confusion

Two separate, unrelated issues combined to make the switch's management address look unstable during setup:

1. **A DHCP Reservation set on the router had no effect.** Root cause, found by opening the switch's own IP Address Setting page: the switch's `DHCP Setting` was already `Disable` — it had a locally configured **static** IP the whole time and was never making a DHCP request, so the router-side reservation had nothing to attach to.
2. **A temporary IP collision.** While experimenting with the switch's static address, it was briefly set to the same IP a laptop on the network was already using (both landed on `<ADMIN_SUBNET>.102`). Browsing to the "reserved" address showed the laptop, not the switch, and vice versa depending on ARP cache state.

**Resolution:** moved the switch's static management IP into the VLAN 1 range, out of any DHCP pool entirely, and moved the laptop off the address that had collided with it.

## Same PVID bug recurred when adding a second server

**Symptom:** the Kubernetes worker node (`<WORKER_HOSTNAME>`), connected to Port 5, got an IPv6 link-local address but no IPv4 lease.

**Root cause:** VLAN membership was set correctly (Untagged, VLAN 20), but the port's PVID had been left at its default of `1`.

**Fix:** `802.1Q PVID Setting → Port 5 → 20`. The check has to be repeated for *every* new access port, not just the first few configured. Server-side view of the same incident: [SRV-06-TRBL](../server/SRV-06-TRBL-Second-Node-Setup.md#no-ipv4-address-despite-a-healthy-link).

## Related

- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md) — the working configuration
- Guide: [VLANs, Trunk vs. Access, PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md)
