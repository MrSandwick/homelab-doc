---
tags: [homelab-project, homelab, note, project, networking, isp]
---

# Internet Uplink

## Starting point

ISP-supplied gateway — a combined modem/router/Wi-Fi unit. Its own status page reports `Connection Type: Cellular`, confirming this is a cellular-based 5G Home Internet product rather than a fiber/cable connection. Out of the box it runs its own NAT, DHCP, and Wi-Fi, all of which needed to be bypassed or disabled so the Omada router could take over as the "real" router for the home network.

## Attempted: IP Passthrough

IP Passthrough — bridging the ISP gateway so a downstream router receives the public IP directly, avoiding a second layer of NAT — is documented by the ISP as a supported feature on their Fios-class gateways. On the specific 5G Home Internet gateway and firmware version installed here:

- The setting does not appear anywhere in the ISP's mobile app, including its advanced/network settings sections
- The gateway's own local web UI exposes a **read-only** Broadband Connection status page (WAN IP, subnet mask, default gateway, DNS, packet counters) but no editable Passthrough toggle anywhere in the Network Settings menu tree
- Contacting ISP support to have it enabled server-side was identified as the remaining option but was not pursued for this project — see the decision below

## Decision: Double NAT

> New to NAT, or unsure why running two of them in a row is usually fine? See [Guide-Double-NAT-and-IP-Passthrough](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Double-NAT-and-IP-Passthrough.md) for the plain-language version.

Rather than spend more time chasing a Passthrough toggle that may not exist in this firmware/hardware combination, the ISP gateway was left in its default routing mode and the Omada router's WAN port was connected to one of the gateway's LAN ports.

```
ISP gateway (own NAT, own DHCP)
        |
   LAN port -> ER605 WAN port
        |
   ER605 (own NAT, VLANs, DHCP, firewall)
        |
   Switch -> Access Point -> clients
```

**Trade-off accepted:** double NAT complicates inbound port-forwarding from the public internet — it would require forwarding rules on *both* the ISP gateway and the ER605 for anything to be reachable from outside. This project has no inbound-from-internet requirement (no self-hosted service needs to be reachable from outside the home network), so the trade-off costs nothing in practice. If that changes later, a VPN back into the Admin VLAN is the planned alternative to opening inbound ports at all.

## ER605 WAN configuration

```
Connection Type: Dynamic IP (DHCP)
```

No static WAN fields were needed — the ER605 picks up an address from the ISP gateway's own DHCP pool automatically, same as any other client behind it.

## Follow-up: disabling the ISP gateway's own Wi-Fi

To avoid two parallel Wi-Fi networks broadcasting in the same home, the ISP gateway's own Wi-Fi radios were turned off once the Omada AP was confirmed working (ISP app → Wi-Fi Settings → disable both the 2.4 GHz and 5 GHz radios), leaving the EAP610 as the only access point in the house.

## A related ISP behavior discovered later: blocking public DNS resolvers

This ISP also blocks direct connections to well-known public DNS resolvers (`8.8.8.8`, `1.1.1.1`) by IP — discovered and worked around at the server level, not the network level. The fix applied is pointing each node's DNS at the router (`<GATEWAY_IP>`) instead of the blocked resolvers; DNS-over-TLS was investigated as an alternative but not carried through. Full diagnosis and both approaches documented in the [server network configuration](../server/SRV-03-Network-Configuration.md#real-troubleshooting-isp-blocking-public-dns-resolvers). Noted here because it's the same ISP, and worth knowing about alongside the IP Passthrough limitation above if this network is ever rebuilt against a different ISP connection type.

## Related

- [Project Overview](./NET-00-Project-Overview.md)
- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md)
- Server docs: [Network Configuration](../server/SRV-03-Network-Configuration.md) — the DNS-blocking troubleshooting for this ISP
- Guide: [Double NAT and IP Passthrough](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Double-NAT-and-IP-Passthrough.md)
