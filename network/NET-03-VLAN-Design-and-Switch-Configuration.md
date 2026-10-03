---
tags: [homelab-project, homelab, note, project, networking, vlan]
---

# VLAN Design and Switch Configuration

> New to VLANs, trunk vs. access ports, or tagged vs. untagged traffic? See [Guide-VLANs-Trunk-Access-PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md) for the plain-language version before diving into the config below.

Problems hit during this work are recorded separately in [NET-03-TRBL](./NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md).

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

**PVID:** every access port's PVID (`VLAN → 802.1Q PVID Setting`) is set to the same VLAN as its Untagged membership — ports 5, 6 and 7 → PVID `20`. Membership alone is not sufficient: PVID decides which VLAN untagged incoming traffic from the connected device is placed into. The check is repeated for every new access port — see [NET-03-TRBL](./NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md#vlan-membership-set-pvid-left-at-default).

**Switch management IP:** static, configured on the switch itself (`DHCP Setting: Disable`), in the VLAN 1 range and outside every DHCP pool — a router-side DHCP reservation has no effect on it. See [NET-03-TRBL](./NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md#switch-management-ip-confusion).

## Related

- [NET-03-TRBL](./NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md) — troubleshooting for this doc
- [Hardware Selection](./NET-01-Hardware-Selection.md) — the switch web-UI limitation referenced above
- [Router Configuration](./NET-04-Router-Configuration.md)
- Guide: [VLANs, Trunk vs. Access, PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md)
