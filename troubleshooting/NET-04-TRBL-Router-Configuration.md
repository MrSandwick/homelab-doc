---
tags: [homelab-project, homelab, note, project, networking, router, troubleshooting]
---

# Troubleshooting — Router Configuration (ER605)

> Companion to [Router Configuration (ER605)](../network/NET-04-Router-Configuration.md), which records only the working configuration. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Relates to |
|---|---|
| [Standalone config not preserved on adoption](#standalone-config-not-preserved-on-adoption) | [Management path](../network/NET-04-Router-Configuration.md#management-path) |
| [Controller connectivity lost after an SSID change](#controller-connectivity-lost-after-an-ssid-change) | [Controller management access](../network/NET-04-Router-Configuration.md#controller-management-access) |
| [Changing the Management VLAN made connectivity worse](#changing-the-management-vlan-made-connectivity-worse) | [Controller management access](../network/NET-04-Router-Configuration.md#controller-management-access) |

## Standalone config not preserved on adoption

**What happened:** when the router was adopted into the Omada Controller, the VLANs already created through the standalone web UI **did not carry over** — the controller's `Settings → Wired Networks → LAN` showed only the factory-default VLAN 1. Adoption appears to push the controller's own (empty/default) network configuration onto the device rather than importing what the device already had.

**Fix:** VLANs 10 and 20 were recreated from scratch inside the controller (`Settings → Wired Networks → LAN → Add`) with the same subnets as before.

**Takeaway:** decide on one management path — standalone *or* controller — per device before doing real configuration work on it, or plan to redo any standalone-side setup once the device is adopted.

## Controller connectivity lost after an SSID change

**Symptom:** after editing SSID security settings on the access point (see [Wireless / RADIUS Integration](../network/NET-06-Wireless-RADIUS-Integration.md)), both the router and the AP dropped to `DISCONNECTED` / `ADOPT FAILED` in the controller's device list, while the network itself — routing, DHCP, internet access — kept working normally the entire time.

**Diagnosis:**

```
ping <ROUTER_MGMT_IP>
# succeeded

Test-NetConnection <ROUTER_MGMT_IP> -Port 29814
# failed — this is one of Omada's device-management ports
```

The controller-management ports were unreachable specifically **when tested from a device sitting in VLAN 20**, even though basic ICMP traffic crossed VLANs fine. The same result was reproduced for both the router and the access point, and for a few other ports in the Omada management range.

**Fix:** connected a laptop directly to a VLAN 1 (native) switch port and, from there, ran the Omada Controller's **Force Provision** action against each affected device (a temporary disconnect followed by a full config re-sync — distinct from **Forget**, which factory-resets the device and discards its configuration entirely). Both devices returned to `Connected` shortly after.

**Takeaway:** manage the Omada Controller from a device physically on the native/management VLAN, not from the Admin VLAN — in this setup, the controller's device-management protocol did not traverse the inter-VLAN routing path even where general ICMP/HTTP traffic did. Worth re-testing after building out proper ACLs (see [Access Control and Isolation](../network/NET-07-Access-Control-and-Isolation.md)), since it is not yet clear whether this was routing-related or a firewall default.

## Changing the Management VLAN made connectivity worse

**What was tried:** following up on the incident above, the Omada Controller's **Management VLAN** (`Settings → Wired Networks → LAN → VLAN → Management VLAN`) was changed from `Default` to `Custom → VLAN 20` as a permanent fix — the theory being that moving the device-management plane into the VLAN normally used for admin work would remove the cross-VLAN issue, rather than requiring a physical VLAN-1 connection every time.

**Result — worse, not better:**

- The access point's management IP disappeared from **both** VLAN 1 and VLAN 20 (`nmap -sn` scans of both subnets found no trace of its MAC address)
- The router's own management IP appeared to remain reachable, but the AP could not be recovered via **Force Provision** — the device was not answering on any known subnet
- The associated WPA-Enterprise SSID ("SuperAlga") stopped broadcasting entirely

**Recovery:** a full factory reset of the access point (physical reset button), which restored basic connectivity but wiped its entire configuration — SSIDs, RADIUS binding, and VLAN settings all had to be recreated from scratch (see [Wireless / RADIUS Integration](../network/NET-06-Wireless-RADIUS-Integration.md) for the SSID recreation). The simpler WPA-Personal SSID ("MiniAlga") was unaffected by the reset and kept working throughout, suggesting Enterprise/RADIUS-bound SSIDs carry more fragile, harder-to-recover state than simple Personal ones.

**Conclusion:** the Management VLAN setting was reverted to `Default`. The underlying limitation (controller-management ports only reachable from VLAN 1) remains unresolved; the working mitigation is still to physically connect to a VLAN 1 port whenever controller-level device management is needed. Whether each device's `Layer-3 Accessibility` toggle (under `Management Access`) would have solved the original issue without this risk was never tested, since this experiment interrupted that investigation — worth revisiting separately, and only with a verified physical-recovery path in hand.

## Related

- [Router Configuration (ER605)](../network/NET-04-Router-Configuration.md) — the working configuration
- [Wireless / RADIUS Integration](../network/NET-06-Wireless-RADIUS-Integration.md) — the SSID change that triggered the connectivity incident
- [Access Control and Isolation](../network/NET-07-Access-Control-and-Isolation.md) — the ACL work done using the VLAN-1-connection workaround
- Guide: [SDN Controller vs. Standalone](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/operations/Guide-SDN-Controller-vs-Standalone.md)
