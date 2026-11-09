---
tags: [homelab-project, homelab, note, project, networking, security, dmz, planned]
---

# DMZ VLAN for the Public Site — Plan

> Status: ⬜ **Planned, nothing changed yet.** This page records the design and the open decisions before any of it is applied. The public site currently answers from a Gateway inside the cluster on the Admin VLAN — see [Public Port Forwarding](./NET-09-Public-Port-Forwarding.md) and [Public Gateway and HTTPS](../server/SRV-23-Public-Gateway-and-HTTPS.md).

## Contents

1. [Why](#why)
2. [What "move it to a DMZ" actually means here](#what-move-it-to-a-dmz-actually-means-here)
3. [Options](#options)
4. [Proposed VLAN and addressing](#proposed-vlan-and-addressing)
5. [Proposed ACL rule set](#proposed-acl-rule-set)
6. [Migration steps](#migration-steps)
7. [Verification](#verification)
8. [Rollback](#rollback)
9. [Open decisions](#open-decisions)

## Why

Ports 80 and 443 are forwarded from the internet to `<PUBLIC_GW_IP>`, an address that sits in the **Admin VLAN** — the same segment as the cluster nodes, the admin laptop and the management plane. The IDS saw what that exposure attracts within hours of the forward being opened: reputation-list sources, a GraphQL introspection probe, and repeated requests for `/.env`, `phpinfo` and `wp-config` backups ([AI Alert Triage](../server/SRV-27-AI-Alert-Triage.md#case-study-a-new-public-web-server-appears)). Everything was answered with `404`, so nothing was reached — but the position is the problem, not the outcome: anything that does get through starts its life inside the trusted segment.

A DMZ is the standard answer: put the internet-facing thing in its own segment that may reach the internet but not the rest of the network.

## What "move it to a DMZ" actually means here

The usual DMZ recipe assumes the public service is a machine, so you move that machine to another VLAN. Here it is not a machine — it is a `Gateway` object in Kubernetes, answered by an Envoy pod scheduled somewhere on the cluster, reachable on a MetalLB address advertised by whichever node currently owns it.

That distinction decides how much isolation each option actually buys:

- Moving only the **address** changes where the traffic arrives, but the pod still runs on a node in the Admin VLAN. A container escape would land on an Admin-VLAN host. The VLAN boundary constrains the traffic path, not the blast radius.
- Moving the **node** is what a real DMZ means: the machine that executes untrusted input is in the isolated segment, and the ACL is between it and everything else.

Writing this down first matters because option A below looks like a DMZ on a diagram and is not one.

## Options

| Option | What moves | Isolation gained | Cost |
|---|---|---|---|
| **A. DMZ address only** | A tagged sub-interface in the DMZ VLAN on the nodes, plus a second MetalLB pool in the DMZ subnet; the `public` Gateway takes its address from that pool | Inbound internet traffic no longer transits the Admin subnet; ACLs can restrict the DMZ address. Pod still executes on an Admin-VLAN node | Low — a switch port change, a sub-interface, one MetalLB pool |
| **B. Dedicated DMZ node** | A node placed in the DMZ VLAN; the public workload pinned to it with a taint and node affinity | Real segmentation: the host that runs internet-facing code sits in the isolated segment | Higher — a node leaves the cluster's normal segment, so control-plane traffic must cross the ACL (see [Open decisions](#open-decisions)) |
| **C. No VLAN change** | Nothing; rely on Kubernetes `NetworkPolicy` and the existing Gateway rate limits | No new VLAN to maintain | Does not address the question asked; recorded so the "do nothing" baseline is explicit |

**Leaning:** B is the only one that matches what the IDS case study was worried about. A is a reasonable first step if B's cluster-traffic question turns out to be awkward, provided the docs say plainly that it is a partial measure.

## Proposed VLAN and addressing

Following the existing scheme in [VLAN Design](./NET-03-VLAN-Design-and-Switch-Configuration.md), where VLAN 1 is management, 10 is Users and 20 is Admin:

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| `<DMZ_VLAN_ID>` (30 proposed) | DMZ | `<DMZ_SUBNET>/24` | Internet-facing services only |

- DHCP pool starting at `.100`, as on the other VLANs, with `.2–.99` left for fixed addresses.
- The public Gateway address moves from `<PUBLIC_GW_IP>` to an address in `<DMZ_SUBNET>`; the ER605 forwarding rule in [Public Port Forwarding](./NET-09-Public-Port-Forwarding.md) is repointed to it.
- The switch port for the DMZ member is tagged for the new VLAN with a matching PVID — the PVID step is the one that has bitten this network twice already ([NET-03-TRBL](../troubleshooting/NET-03-TRBL-VLAN-Design-and-Switch-Configuration.md)).

## Proposed ACL rule set

Gateway ACL, evaluated top-down as in [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md):

```
1. Permit | Source: Network:DMZ   | Destination: WAN              | All protocols
2. Permit | Source: Network:Admin | Destination: Network:DMZ      | All protocols
3. Deny   | Source: Network:DMZ   | Destination: Network:Admin    | All protocols
4. Deny   | Source: Network:DMZ   | Destination: Network:Users    | All protocols
5. Deny   | Source: Network:DMZ   | Destination: Network:Default  | All protocols
```

Rule 2 keeps administration working from the Admin side; rules 3–5 are the point of the exercise. Order matters, and a leftover broad Permit above them would silently cancel them — exactly the mistake recorded in [NET-07-TRBL](../troubleshooting/NET-07-TRBL-Access-Control-and-Isolation.md#two-gateway-acl-rule-authoring-mistakes).

## Migration steps

1. Create the VLAN on the ER605 (subnet, DHCP pool) and on the switch (802.1Q membership **and** PVID on the member port).
2. Put the DMZ member in place — a sub-interface (option A) or the node itself (option B).
3. Add the ACL rules above, with the Deny rules in place **before** the service is exposed.
4. Repoint the ER605 port-forward to the new address; leave the ISP gateway rule untouched (it targets the ER605's WAN address, which does not change).
5. Re-issue or confirm the Let's Encrypt certificate — HTTP-01 validation must reach the new address ([Public Gateway and HTTPS](../server/SRV-23-Public-Gateway-and-HTTPS.md)).
6. Add the DMZ subnet to the IDS device inventory and to ntopng's local-network list, so alerts from it are attributed rather than flagged as unknown ([AI Alert Triage](../server/SRV-27-AI-Alert-Triage.md), [Traffic Monitoring](./NET-08-Monitoring.md)).

## Verification

```
# from a DMZ host: internet yes, internal no
ping 8.8.8.8                      # expect: replies
ping <ADMIN_HOST_IP>              # expect: timeout
curl -m 5 http://<ROUTER_MGMT_IP> # expect: no answer

# from an Admin host: DMZ reachable (rule 2)
curl -m 5 http://<DMZ_SUBNET>.x

# from outside (phone on cellular data)
curl -I https://<DOMAIN>          # expect: the site, with a valid certificate
```

The mirror port already copies the router-uplink trunk, so DMZ traffic will appear in Suricata and ntopng without extra configuration — confirm that it does, since a silent sensor and a correctly isolated segment look identical from the alert log.

## Rollback

Each step is reversible on its own, and the riskiest one is not the ACL but the address change: if the forward points at an address nothing answers on, the site is down from the internet while still looking healthy from inside. Keep the old `<PUBLIC_GW_IP>` reachable until the new path is confirmed from outside, then remove it.

The earlier Management VLAN experiment is the cautionary tale here — a VLAN change that looked reasonable left the access point unreachable on every VLAN and required a physical factory reset ([NET-04-TRBL](../troubleshooting/NET-04-TRBL-Router-Configuration.md)). Do not apply any of this without physical access to the gear.

## Open decisions

- [ ] **Option A or B.** B is the real DMZ; A is a partial measure. Decide before any switch change.
- [ ] **If B: how does the DMZ node stay in the cluster?** `kubelet` → API server, etcd peers and the CNI overlay all have to cross the DMZ boundary, which means punching holes in exactly the ACL that justifies the move. Worth checking whether a single-node exception (DMZ → Admin on the API-server port only) is acceptable, or whether the public workload should leave Kubernetes entirely.
- [ ] **MetalLB advertisement mode.** L2 mode requires the advertising node to have an interface in the DMZ subnet. Confirm whether a tagged sub-interface is enough or whether this pushes toward BGP.
- [ ] **Is there a second public host?** The IDS case study mentions a friend's site; the documented public path is the cluster Gateway. If a separate machine exists, it is the first thing that should move.
- [ ] **VLAN id and subnet** — 30 and `<DMZ_SUBNET>` are proposals, not decisions.

## Related

- [Public Port Forwarding](./NET-09-Public-Port-Forwarding.md) — the forwarding rules this plan repoints
- [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md) — the ACL model the rules above follow
- [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md) — the existing VLAN scheme
- Server docs: [Public Gateway and HTTPS](../server/SRV-23-Public-Gateway-and-HTTPS.md) — what actually answers on the public address
- Server docs: [AI Alert Triage](../server/SRV-27-AI-Alert-Triage.md) — the scanning that prompted this
- Guide: [Network Isolation vs. ACLs](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/segmentation/Guide-Network-Isolation-vs-ACLs.md)
