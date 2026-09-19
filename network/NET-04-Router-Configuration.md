---
tags: [homelab-project, homelab, note, project, networking, router]
---

# Router Configuration (ER605)

## Management path

> Unfamiliar with the standalone-vs-controller distinction, or what "adopting" a device means? See [Guide-SDN-Controller-vs-Standalone](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-SDN-Controller-vs-Standalone.md).

Configured first via the router's own standalone web UI (`<ROUTER_MGMT_IP>`), then migrated to the Omada SDN Controller once the switch and access point were also brought under controller management, so all three devices could eventually be administered from one place.

## Real troubleshooting: standalone config is not preserved on adoption

When the router was adopted into the Omada Controller, the VLANs that had already been created through the standalone web UI **did not carry over** — the controller's `Settings → Wired Networks → LAN` showed only the factory-default VLAN 1. Adoption appears to push the controller's own (empty/default) network configuration onto the device rather than importing whatever the device already had configured.

**Practical takeaway:** decide on one management path — standalone *or* controller — per device before doing real configuration work on it, or plan to redo any standalone-side setup once the device is adopted. In this case, VLANs 10 and 20 were simply recreated from scratch inside the controller (`Settings → Wired Networks → LAN → Add`) with the same subnets as before.

## LAN / VLAN configuration (final, controller-managed)

```
Users  (VLAN 10): <USERS_SUBNET>/24, DHCP <USERS_SUBNET>.100–.200
Admin  (VLAN 20): <ADMIN_SUBNET>/24, DHCP <ADMIN_SUBNET>.100–.200
```

**Network Isolation** (Omada's per-LAN client-to-client isolation setting) is enabled on VLAN 10 and left disabled on VLAN 20, since the server and admin devices on VLAN 20 need to reach each other. See [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md) for what this does and doesn't cover.

## Real troubleshooting: losing controller connectivity after an SSID change

After editing SSID security settings on the access point (see [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md)), both the router and the AP dropped to `DISCONNECTED` / `ADOPT FAILED` in the controller's device list, while the network itself — routing, DHCP, internet access — kept working normally the entire time.

**Diagnosis:**

```
ping <ROUTER_MGMT_IP>
# succeeded

Test-NetConnection <ROUTER_MGMT_IP> -Port 29814
# failed — this is one of Omada's device-management ports
```

The controller-management ports were unreachable specifically **when tested from a device sitting in VLAN 20**, even though basic ICMP traffic crossed VLANs fine. The same result was reproduced for both the router and the access point, and for a few other ports in the Omada management range.

**Fix:** connected a laptop directly to a VLAN 1 (native) switch port and, from there, ran the Omada Controller's **Force Provision** action against each affected device (a temporary disconnect followed by a full config re-sync — distinct from **Forget**, which factory-resets the device and discards its configuration entirely). Both devices returned to `Connected` shortly after.

**Takeaway:** manage the Omada Controller from a device physically on the native/management VLAN, not from the Admin VLAN — in this setup, the controller's own device-management protocol did not traverse the inter-VLAN routing path even where general ICMP/HTTP traffic did. This is worth re-testing after building out proper ACLs (see [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md)), since it's not yet clear whether this was routing-related or a firewall default.

## Real troubleshooting: changing the Management VLAN made connectivity worse

Following up on the controller-connectivity issue above, changing the Omada Controller's **Management VLAN** (`Settings → Wired Networks → LAN → VLAN → Management VLAN`, from `Default` to `Custom → VLAN 20`) was attempted as a permanent fix — the theory being that moving the device-management plane into the same VLAN normally used for admin work would eliminate the cross-VLAN routing issue entirely, rather than requiring a physical VLAN-1 connection every time.

**Result: this made the problem worse, not better.** After applying the change:

- The access point's management IP disappeared from **both** VLAN 1 and VLAN 20 (`nmap -sn` scans of both subnets found no trace of its MAC address)
- The router's own management IP appeared to remain reachable, but the AP could not be recovered via **Force Provision** — the device simply wasn't answering on any known subnet
- The associated WPA-Enterprise SSID ("SuperAlga") stopped broadcasting entirely

**Recovery required a full factory reset** of the access point (physical reset button), which restored basic connectivity but wiped its entire configuration — SSIDs, RADIUS binding, and VLAN settings all had to be recreated from scratch (see [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md) for the SSID recreation). The simpler WPA-Personal SSID ("MiniAlga") was unaffected by the reset and kept working throughout, suggesting Enterprise/RADIUS-bound SSIDs carry more fragile, harder-to-recover state than simple Personal ones.

**Conclusion:** the Management VLAN setting was reverted to `Default`. The underlying limitation (controller-management ports only reachable from VLAN 1, documented above) remains unresolved — the working mitigation continues to be **physically connecting to a VLAN 1 port whenever controller-level device management is needed**, rather than changing where the management plane itself lives. Whether the router/switch/AP's individual `Layer-3 Accessibility` toggle (seen under each device's `Management Access` settings) would have solved the original issue without this risk was never tested, since this experiment interrupted that investigation — worth revisiting separately, and only with a verified physical-recovery path in hand before changing anything.

## Related

- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md)
- [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md) — the SSID change that triggered the incident above
- [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md) — the ACL work done using the VLAN-1-connection workaround
- Guides: [SDN Controller vs. Standalone](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-SDN-Controller-vs-Standalone.md) · [Network Isolation vs. ACLs](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Network-Isolation-vs-ACLs.md)
