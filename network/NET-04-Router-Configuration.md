---
tags: [homelab-project, homelab, note, project, networking, router]
---

# Router Configuration (ER605)

Problems hit during this work are recorded separately in [NET-04-TRBL](./NET-04-TRBL-Router-Configuration.md).

## Management path

> Unfamiliar with the standalone-vs-controller distinction, or what "adopting" a device means? See [Guide-SDN-Controller-vs-Standalone](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-SDN-Controller-vs-Standalone.md).

Configured first via the router's own standalone web UI (`<ROUTER_MGMT_IP>`), then migrated to the Omada SDN Controller once the switch and access point were also brought under controller management, so all three devices could eventually be administered from one place.

Standalone configuration is not carried over on adoption: VLANs 10 and 20 were recreated inside the controller (`Settings → Wired Networks → LAN → Add`) with the same subnets — see [NET-04-TRBL](./NET-04-TRBL-Router-Configuration.md#standalone-config-not-preserved-on-adoption).

## LAN / VLAN configuration (final, controller-managed)

```
Users  (VLAN 10): <USERS_SUBNET>/24, DHCP <USERS_SUBNET>.100–.200
Admin  (VLAN 20): <ADMIN_SUBNET>/24, DHCP <ADMIN_SUBNET>.100–.200
```

**Network Isolation** (Omada's per-LAN client-to-client isolation setting) is enabled on VLAN 10 and left disabled on VLAN 20, since the server and admin devices on VLAN 20 need to reach each other. See [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md) for what this does and doesn't cover.

## Controller management access

- Controller-level device management (Adopt, Force Provision) is done from a device physically connected to a VLAN 1 (native) switch port. Omada's device-management ports (e.g. `29814`) are not reachable from VLAN 20, although ICMP and HTTP cross VLANs normally.
- **Management VLAN** (`Settings → Wired Networks → LAN → VLAN → Management VLAN`) is left at `Default`.

Both points come out of two incidents — see [NET-04-TRBL](./NET-04-TRBL-Router-Configuration.md#controller-connectivity-lost-after-an-ssid-change). Open: whether the per-device `Layer-3 Accessibility` toggle removes the VLAN 1 requirement has not been tested.

## Related

- [NET-04-TRBL](./NET-04-TRBL-Router-Configuration.md) — troubleshooting for this doc
- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md)
- [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md) — the SSID change that triggered the controller-connectivity incident
- [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md) — the ACL work done using the VLAN-1-connection workaround
- Guides: [SDN Controller vs. Standalone](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-SDN-Controller-vs-Standalone.md) · [Network Isolation vs. ACLs](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Network-Isolation-vs-ACLs.md)
