---
tags: [homelab-project, homelab, note, overview, project, networking]
---

# Network — Project Overview

## Goal

Build the network layer for a home lab: a segmented, RADIUS-authenticated network that (a) protects a Kubernetes homelab server behind an "Admin" VLAN, (b) gives everyday household devices their own isolated "Users" network, and (c) demonstrates practical network engineering / security skills for a portfolio, while doubling as the actual production network for the household.

## Requirements driving the design

- Wi-Fi devices on the admin network should authenticate by MAC address against a central RADIUS server rather than a single shared PSK
- Two-tier access: an **Admin** segment (server, admin laptop, admin Wi-Fi) and a **Users** segment (everyday devices), isolated from each other
- The RADIUS server should live on the same physical machine as the Kubernetes homelab (see the [server documentation](../server/SRV-README.md)), reachable only from the Admin segment
- No dependency on the ISP-supplied gateway beyond raw internet access

## Key architectural decisions

| Decision | Choice | Rationale |
|---|---|---|
| ISP integration | Double NAT (ISP gateway kept in place, own router behind it) | IP Passthrough is documented as supported on the ISP's Fios-class gateways, but the toggle is missing from both the mobile app and the local web UI on the cellular 5G Home Internet gateway/firmware actually installed. Double NAT has no practical downside here since there's no inbound port-forwarding requirement. See [Internet Uplink](./NET-02-Internet-Uplink.md) |
| VLAN segmentation | 3 VLANs: 1 (native/mgmt), 10 (Users), 20 (Admin) | Keeps device-management traffic (router/switch/AP web UIs) on a separate broadcast domain from both user and admin client traffic. New to VLANs? See [Guide-VLANs-Trunk-Access-PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md) |
| RADIUS authentication method | WPA2-Enterprise (PEAP/MSCHAPv2), device MAC address used as both identity and password | Chosen after two other Omada authentication modes were evaluated (PPSK-with-RADIUS, and MAC-Based Authentication with an empty password); WPA2-Enterprise was the first verified working end-to-end. See [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md) and [Guide-WiFi-Security-Modes](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-WiFi-Security-Modes.md) |
| Device management | Standalone web UI initially, migrated to Omada SDN Controller | Controller-based management is required to expose MAC-Based Authentication and to manage SSID/VLAN config across all three devices from one place |

## Hardware summary

See [Hardware Selection](./NET-01-Hardware-Selection.md) for the full comparison.

- **Router:** TP-Link Omada ER605
- **Switch:** TP-Link TL-SG108E (managed, 8-port)
- **Access point:** TP-Link EAP610 (Wi-Fi 6)
- **RADIUS + application server:** the same machine used for the Kubernetes homelab (see the [server documentation](../server/SRV-README.md)) — `<HOSTNAME>`, `<SERVER_IP>` in VLAN 20

## Status at time of writing

- ✅ ISP gateway bridged into a double-NAT chain; own router providing DHCP/routing/VLANs
- ✅ VLAN 10 (Users) and VLAN 20 (Admin) created and correctly tagged across the router↔switch trunk
- ✅ FreeRADIUS installed and running on the server, MAC-based `users` file populated
- 🟡 WPA2-Enterprise Wi-Fi (verified working on Android and Windows earlier) regressed after the access point was factory-reset during an unrelated Management VLAN experiment (see [Router Configuration](./NET-04-Router-Configuration.md)) — the SSID needs to be recreated using the already-documented recipe; see [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md)
- ✅ Client-to-client isolation confirmed on the Users VLAN; Users VLAN confirmed unable to reach the Admin VLAN or any device-management IP
- 🟡 Login-free MAC-Based Authentication (Omada's dedicated feature, `Empty Password: Enable`) configured but not yet verified end-to-end — a MAC-address-format mismatch between this feature and the existing RADIUS `users` file needs to be resolved first
- ✅ Fine-grained management ACLs implemented on the real hardware (Omada Gateway ACL) — only specific admin devices (by fixed IP) can reach router/switch/server management interfaces; verified working. See [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md)
- ✅ Traffic monitoring verified working — ntopng (with Redis, via docker-compose) showing live per-device traffic across all VLANs through switch port mirroring. See [Traffic Monitoring](./NET-08-Monitoring.md)
- 🔴 An attempt to change the Omada Controller's Management VLAN (from the default/native VLAN to the Admin VLAN, as a permanent fix for the controller-connectivity issue below) made things worse rather than better — the access point became unreachable on every VLAN and required a factory reset to recover. Reverted; not adopted as a fix. See [Router Configuration](./NET-04-Router-Configuration.md)
- ⬜ IDS/IPS (Suricata/Zeek) and LLM-assisted alert triage — not started

Continue at [Hardware Selection](./NET-01-Hardware-Selection.md) for the equipment comparison, or jump straight to [FreeRADIUS Installation](./NET-05-FreeRADIUS-Installation.md) for the authentication work.

## A parallel exercise: Cisco Packet Tracer lab

Before/alongside the real deployment, the same VLAN + RADIUS + ACL design was rebuilt from scratch in Cisco Packet Tracer (a `2911` router, a `2960-24T` switch, a simulated AAA server) purely as a portfolio/interview-prep exercise. It is **not** part of this production network and is not documented here; it exercised the same concepts — VLAN trunking, router-on-a-stick inter-VLAN routing, RADIUS AAA for device-management logins, and standard/extended ACLs restricting management access to specific hosts — using Cisco IOS CLI syntax rather than TP-Link Omada's web UI. Where the real network hasn't yet replicated something the Packet Tracer lab already covers, that's called out explicitly in the relevant doc rather than left implicit — the management ACLs that were originally only a Packet Tracer exercise have since been implemented on the real hardware, see [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md).

## Related

- [Hardware Selection](./NET-01-Hardware-Selection.md)
- Server documentation: [SRV-README](../server/SRV-README.md) — the server this network protects
- Guides: [VLANs, Trunk vs. Access, PVID](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-VLANs-Trunk-Access-PVID.md) · [RADIUS and AAA](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-RADIUS-and-AAA.md) · [SDN Controller vs. Standalone](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-SDN-Controller-vs-Standalone.md) · [Network Isolation vs. ACLs](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Network-Isolation-vs-ACLs.md)
