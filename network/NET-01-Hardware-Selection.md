---
tags: [homelab-project, homelab, note, project, hardware, networking]
---

# Network — Hardware Selection

## Constraints

- Budget-conscious, iterative: roughly $150–200 targeted for router + switch + AP combined
- Wanted a single vendor ecosystem so VLANs, RADIUS profiles, and SSIDs could eventually be managed from one controller instead of three separate device UIs
- Server hardware is shared with / reused from the Kubernetes homelab build and is not re-selected here — see the [server hardware-selection doc](../server/SRV-01-Hardware-Selection.md) for that process. In this section it's simply "the Admin-VLAN server."

## Options evaluated

| Component | Options considered | Rejected because | Chosen |
|---|---|---|---|
| Router | Keep the ISP gateway as the router; TP-Link Omada ER605 | The ISP gateway has no VLAN support and only very basic firewall rules | **ER605** |
| Switch | TP-Link TL-SG108 (unmanaged); TL-SG108E (managed) | The plain **SG108** has no VLAN tagging and no port mirroring at all — its product listing is easy to confuse with the managed **SG108E ("Easy Smart")** from the title alone; the spec sheets had to be compared line-by-line before ordering | **TL-SG108E** |
| Access point | TP-Link EAP225-Outdoor; TP-Link EAP610 | The EAP225-Outdoor is Wi-Fi 5 only and its weatherproof outdoor housing isn't needed for an indoor deployment | **EAP610** (Wi-Fi 6 / AX1800, indoor, WPA2/3-Enterprise support) |

## Final decision

- **Router — TP-Link Omada ER605.** 1× WAN, 2× WAN/LAN (configurable), 2× LAN, 1× USB. Supports VLAN, ACL, multi-net DHCP, policy-based firewall, and integrates with the Omada SDN Controller. ~$50.
- **Switch — TP-Link TL-SG108E.** 8-port Gigabit "Easy Smart" managed switch: 802.1Q VLAN, port mirroring, PVID per port. ~$25–30.
- **Access point — TP-Link EAP610.** Wi-Fi 6 (AX1800), 2.4 GHz + 5 GHz, PoE+ input with a standard power adapter also included in the box (no separate PoE injector needed), WPA2/3-Personal and -Enterprise. ~$70–90.

## A firmware/UI limitation discovered post-purchase

The TL-SG108E's built-in web UI (browsing directly to its management IP) is unreliable on modern browsers: requests return `ERR_EMPTY_RESPONSE` or `ERR_CONNECTION_TIMED_OUT` even while the device answers ICMP pings and the TCP port is confirmed open (`Test-NetConnection ... -Port 80` reports `TcpTestSucceeded: True`). This is a known limitation of this switch's embedded web server, not a network fault — restarting the switch, changing browsers, and re-checking DHCP/ARP state all made no difference.

**Workaround used throughout this project:** TP-Link's standalone **Easy Smart Configuration Utility** (downloaded from the product's support/download page) instead of a browser for any configuration on this switch. It discovers the switch on the local network directly and isn't affected by the broken embedded web server. See [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md) for the actual VLAN/PVID/mirroring work done through it.

## Server

The RADIUS server and Docker host run on the same machine used for the Kubernetes homelab — see the [server hardware-selection doc](../server/SRV-01-Hardware-Selection.md) for that selection process. In this section it appears only by its role: "the Admin-VLAN server" at `<SERVER_IP>`.

## Related

- [Project Overview](./NET-00-Project-Overview.md)
- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md)
- Guides: [VLANs, Trunk vs. Access, PVID](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-VLANs-Trunk-Access-PVID.md) · [SDN Controller vs. Standalone](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-SDN-Controller-vs-Standalone.md)
