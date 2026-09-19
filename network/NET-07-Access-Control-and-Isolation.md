---
tags: [homelab-project, homelab, note, project, networking, security]
---

# Access Control and Isolation

> VLANs, Network Isolation, and ACLs sound similar but answer different questions — see [Guide-Network-Isolation-vs-ACLs](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-Network-Isolation-vs-ACLs.md) if the distinction below isn't obvious.

## What's implemented on the real network

Omada's built-in **Network Isolation** (a per-LAN setting under `Settings → Wired Networks → LAN → <network>`) is enabled on VLAN 10 (Users) and left disabled on VLAN 20 (Admin). This blocks client-to-client traffic *within* the Users VLAN (e.g. one guest device can't reach another), and — combined with VLAN separation itself — prevents Users-VLAN devices from reaching Admin-VLAN devices at all.

Verified with a direct ping test from a device on the Users SSID toward a known Admin-VLAN host:

```
ping <ADMIN_HOST_IP>
# Request timed out.        <- expected / desired result

ping 8.8.8.8
# succeeds — confirms isolation is scoped to internal traffic, not blocking internet access
```

Management-plane addresses (router/switch/AP web UIs, all on VLAN 1) were also confirmed unreachable from the Users VLAN, both by `ping` and by attempting to load the router's web UI in a browser from a Users-VLAN device.

## What's implemented on the real network (management ACLs)

The original design goal — *only specific admin devices, identified by IP, may reach the router/switch/server management interfaces; every other device on the Admin VLAN is blocked even though it shares the same subnet* — is now implemented on the real Omada hardware, not just in the parallel Packet Tracer lab.

**Where:** Omada splits ACLs by device type (`Gateway ACL`, `Switch ACL`, `EAP ACL`). Inter-VLAN restrictions belong in **Gateway ACL**, since that's enforced by the router.

**Prerequisite — fixed IPs for trusted devices:** ACL rules match on IP address, so each trusted admin device first got a **Fixed IP Address** (`Clients → [device] → Use Fixed IP Address`) within its normal VLAN's DHCP pool, rather than relying on a DHCP-assigned address that could change.

**Final rule set (Gateway ACL), in order:**

```
1. Permit | Source: IP Group [trusted admin devices] | Destination: Network:Default | All protocols
2. Deny   | Source: Network:Admin                    | Destination: Network:Default | All protocols
```

Order matters — Omada evaluates rules top-down and stops at the first match, so the Permit rule for trusted devices must sit above the general Deny.

### Real troubleshooting: two rule-authoring mistakes caught before they caused a lockout or a leak

1. **Action/name mismatch.** The first version of the Permit rule was named `Permit-to-Management` but had its **Action** field still set to `Deny` — which would have blocked the very devices it was meant to allow. Caught by re-reading the rule table rather than assuming the name matched the behavior; fixed by flipping Action to `Permit`.
2. **An overly broad rule silently defeated the narrow one.** A second rule, `Allow-Admin-to-Network` (Source: `Network:Admin` — the *entire* Admin VLAN, not just the trusted IP group; Destination: `Default, Users`; Action: `Permit`), had been created earlier during testing and was still active, positioned **above** the intended Deny rule. Since it matched first, it silently granted management access to every device on the Admin VLAN, not just the trusted ones — making the narrower Permit/Deny pair irrelevant. Found by reviewing the full rule list end-to-end rather than testing only the two rules just written; removed once identified.

**Verified after cleanup:**

```
http://<ROUTER_MGMT_IP>   # from a trusted (fixed-IP) admin device: loads normally
http://<ROUTER_MGMT_IP>   # from any other Admin-VLAN device: unreachable
ping 8.8.8.8              # from either: unaffected — the ACL is scoped to the management subnet only
```

## What's still open

It's still unclear whether the earlier controller-management-port issue (documented in [Router Configuration](./NET-04-Router-Configuration.md)) was routing-related or an Omada firewall default — the investigation into the `Layer-3 Accessibility` toggle as a possible fix was interrupted by the Management VLAN incident and hasn't been resumed. The physical-VLAN-1-connection workaround remains in use for controller-level device management (Force Provision, Adopt, etc.) in the meantime.

## Related

- [Project Overview](./NET-00-Project-Overview.md)
- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md)
- [Router Configuration](./NET-04-Router-Configuration.md)
- Guide: [Network Isolation vs. ACLs](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-Network-Isolation-vs-ACLs.md)
