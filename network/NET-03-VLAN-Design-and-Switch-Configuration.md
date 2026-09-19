---
tags: [homelab-project, homelab, note, project, networking, vlan]
---

# VLAN Design and Switch Configuration

> New to VLANs, trunk vs. access ports, or tagged vs. untagged traffic? See [Guide-VLANs-Trunk-Access-PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md) for the plain-language version before diving into the config below.

## VLAN plan

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 1 | Default / Native | `<MGMT_SUBNET>/24` | Untagged native VLAN; router/switch/AP management IPs live here |
| 10 | Users | `<USERS_SUBNET>/24` | Everyday client devices (phones, guest laptops) |
| 20 | Admin | `<ADMIN_SUBNET>/24` | Server, admin laptop, admin Wi-Fi SSID |

DHCP pools on VLAN 10 and 20 deliberately start at `.100` rather than `.2`, leaving `.2–.99` free for devices that need a fixed address (server, switch, AP), and avoiding the gateway (`.1`) and broadcast (`.255`) addresses.

## Router-side VLAN and trunk configuration (ER605)

The physical LAN port facing the switch was set to trunk (tagged) for both VLANs, with the native VLAN left untagged:

```
Port <SWITCH_UPLINK_PORT>:
  VLAN 1  -> Untagged (native)
  VLAN 10 -> Tagged
  VLAN 20 -> Tagged
```

All other router LAN ports were left untouched (native VLAN only, unused in this deployment).

## Switch-side configuration (TL-SG108E, via Easy Smart Utility)

> Configured through TP-Link's Easy Smart Configuration Utility rather than the switch's browser UI — see the firmware limitation noted in [Hardware Selection](./NET-01-Hardware-Selection.md).

802.1Q VLAN membership:

| Port | Role | VLAN 10 | VLAN 20 |
|---|---|---|---|
| 1 | Uplink to router | Tagged | Tagged |
| 2 | Port mirroring destination (traffic monitoring) | — | — |
| 5 | Second server (`<WORKER_HOSTNAME>`, Kubernetes worker) | — | Untagged |
| 6 | Admin laptop | — | Untagged |
| 7 | Server (`<HOSTNAME>`, control-plane) | — | Untagged |
| 8 | Access point | Tagged | Tagged |
| 3–4 | Unused | — | — |

Port 2 was assigned as the port-mirroring destination *after* Port 5 was needed for the newly added worker node — see [Traffic Monitoring](./NET-08-Monitoring.md) for the mirroring configuration itself.

## Real troubleshooting: VLAN membership vs. PVID

Setting a port's VLAN *membership* to Untagged is not sufficient by itself — the switch also needs the port's **PVID** (Port VLAN ID) set to that same VLAN, because PVID determines which VLAN untagged *incoming* traffic from the connected device gets tagged into on its way through the switch. The membership table alone controls how tagged frames are handled going the other direction.

**Symptom:** a laptop plugged into the port configured for VLAN 20 (server/admin), with correct Untagged membership already set, still received a DHCP lease from VLAN 1 (`<MGMT_SUBNET>.100`, confirmed via `ipconfig`) instead of VLAN 20 as expected.

**Root cause:** the port's PVID was still at its default value of `1`.

**Fix:** `VLAN → 802.1Q PVID Setting`, set the port's PVID to `20`, save, then force the client to re-request an address:

```
ipconfig /release
ipconfig /renew
```

The client immediately obtained a `<ADMIN_SUBNET>.x` address after this change. The same PVID check was subsequently applied to every access port on the switch.

## Real troubleshooting: switch management IP confusion

Two separate, unrelated issues combined to make the switch's management address look unstable during setup:

1. **A DHCP Reservation set on the router had no effect.** Root cause, found by opening the switch's own IP Address Setting page directly: the switch's `DHCP Setting` was already `Disable` — it had a locally configured **static** IP the whole time and was never making a DHCP request in the first place, so the router-side reservation had nothing to attach to.
2. **A temporary IP collision.** While experimenting with the switch's static address, it was briefly set to the same IP a laptop on the network was already using (both landed on `<ADMIN_SUBNET>.102`), producing confusing symptoms — browsing to the "reserved" address showed the laptop, not the switch, and vice versa depending on ARP cache state.

**Resolution:** moved the switch's static management IP into the VLAN 1 range, out of any DHCP pool entirely, and moved the laptop off the exact address that had collided with it.

## Real troubleshooting: same PVID bug recurred when adding a second server

The exact same class of bug documented above recurred when connecting the Kubernetes worker node (`<WORKER_HOSTNAME>`) to Port 5: VLAN membership was set correctly (Untagged, VLAN 20), but the port's PVID had been left at its default of `1`. Confirmed via the new machine getting an IPv6 link-local address but no IPv4 lease at all; fixed the same way — `802.1Q PVID Setting → Port 5 → 20`. Recorded here as a reminder that this check needs to be repeated for *every* new access port, not just the first few configured.

## Related

- [Hardware Selection](./NET-01-Hardware-Selection.md) — the switch web-UI limitation referenced above
- [Router Configuration](./NET-04-Router-Configuration.md)
- Guide: [VLANs, Trunk vs. Access, PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md)
