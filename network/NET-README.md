---
tags: [homelab-project, homelab, moc, networking]
---

# Home Network Build Log — VLAN Segmentation + RADIUS Access Control

A chronological, technical log of building the network layer for a home lab: ISP uplink, router/switch/AP selection, VLAN segmentation, and MAC-based Wi-Fi authentication via FreeRADIUS.

This section documents *what was done and why* — decisions made, exact commands run, and problems encountered along the way (including real troubleshooting sessions, not just the happy path).

## Contents

1. See [Project Overview](./NET-00-Project-Overview.md)
2. See [Hardware Selection](./NET-01-Hardware-Selection.md)
3. See [Internet Uplink](./NET-02-Internet-Uplink.md)
4. See [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md)
5. See [Router Configuration](./NET-04-Router-Configuration.md)
6. See [FreeRADIUS Installation](./NET-05-FreeRADIUS-Installation.md)
7. See [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md)
8. See [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md)
9. See [Traffic Monitoring](./NET-08-Monitoring.md)

## Stack

- **ISP uplink:** cellular 5G Home Internet gateway, kept in place as a bridge/first NAT hop (double NAT — see [Internet Uplink](./NET-02-Internet-Uplink.md))
- **Router:** TP-Link Omada ER605
- **Switch:** TP-Link TL-SG108E (8-port managed "Easy Smart")
- **Access point:** TP-Link EAP610 (Wi-Fi 6, WPA2/3-Enterprise)
- **Authentication:** FreeRADIUS on Ubuntu Server, MAC-address-based
- **Server:** same physical machine as the Kubernetes homelab — see the [server documentation](../server/SRV-README.md)

## A note on placeholders

IP addresses, MAC addresses, hostnames, secrets, and Wi-Fi passwords are replaced with placeholder tokens like `<SERVER_IP>` throughout these docs. See `.env.example` for the full list of placeholders used. Real values are kept locally in a gitignored `.env` file and are never committed.

## New to networking terms used here?

A companion notes vault explains the concepts in plain language, for a reader who doesn't already work in networking — VLANs, RADIUS, Wi-Fi security modes, ACLs, and every real bug hit along the way: **[Homelab Network Guides](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/00-Home-Homelab-Guides.md)**. Each doc below also links out to the specific guide relevant to it.

## Companion section

This network exists to sit in front of a Kubernetes homelab server, documented in the **[server section](../server/SRV-README.md)**. The RADIUS/Docker host referenced throughout this section is that same machine.

A parallel, non-production exercise — rebuilding the same VLAN + RADIUS + ACL design in Cisco Packet Tracer for portfolio/interview purposes — is *not* documented here; see the note in [Project Overview](./NET-00-Project-Overview.md).

## Status

🟡 In progress — core VLAN segmentation and WPA2-Enterprise RADIUS authentication are working and verified. MAC-Based (login-free) Wi-Fi authentication and traffic monitoring are configured but not yet fully verified. See [Project Overview](./NET-00-Project-Overview.md) for the full status table.
