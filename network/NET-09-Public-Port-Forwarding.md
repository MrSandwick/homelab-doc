---
tags: [homelab-project, homelab, note, project, networking, isp, port-forwarding, double-nat, nat]
---

# Public Port Forwarding

> Status: 🟢 **Working.** TCP 80 and 443 from the internet pass through the ISP gateway and the ER605 to the public Gateway in the cluster ([SRV-23](../server/SRV-23-Public-Gateway-and-HTTPS.md)); both were tested from a phone on cellular data. Remote administration, DMZ and UPnP are off. 🟡 Limits: the public IP is dynamic and the uplink is a single 5G connection — see [Limits](#limits).

Picks up after [Internet Uplink](./NET-02-Internet-Uplink.md), where double NAT was accepted with the note that anything reachable from outside would need forwarding rules on **both** routers. This page records those rules. Addresses are placeholders: `<PUBLIC_IP>` is the ISP gateway's WAN address, `<ER605_WAN_IP>` the address the ISP gateway gives the ER605, `<PUBLIC_GW_IP>` the MetalLB address of the public Gateway.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Is the connection reachable at all?](#step-1--is-the-connection-reachable-at-all)
5. [Step 2 — ISP gateway: reserve the ER605's address and forward](#step-2--isp-gateway-reserve-the-er605s-address-and-forward)
6. [Step 3 — ER605: forward to the Gateway](#step-3--er605-forward-to-the-gateway)
7. [Step 4 — Close the side doors](#step-4--close-the-side-doors)
8. [Step 5 — Test from outside](#step-5--test-from-outside)
9. [Verification](#verification)
10. [Limits](#limits)
11. [Files](#files)
12. [Related](#related)

## Components

| Component | Setting | Role |
|---|---|---|
| ISP gateway | DHCP reservation for the ER605; Port Forwarding TCP 80 and 443 → `<ER605_WAN_IP>` | First hop: passes inbound web traffic to the second router |
| ER605 (Omada-managed) | Virtual Servers TCP 80 and 443, WAN → `<PUBLIC_GW_IP>` | Second hop: passes it to the public Gateway in the cluster |
| Public Gateway | MetalLB address `<PUBLIC_GW_IP>` ([SRV-23](../server/SRV-23-Public-Gateway-and-HTTPS.md)) | Terminates TLS and routes to the site |

```
Internet → <PUBLIC_IP>:80/443 → ISP gateway ─forward→ ER605 (<ER605_WAN_IP>) ─forward→ <PUBLIC_GW_IP> (Envoy, VLAN 20)
```

## Why

The portfolio is served from the cluster instead of Firebase. A Cloudflare Tunnel would have avoided any inbound port, but it makes a third-party service part of the path, and avoiding that was the point of leaving Firebase. So the connection itself has to accept inbound web traffic.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Tunnel or open ports | Open ports 80 and 443 | No third-party hop; the ISP gateway reports the same address that the internet sees, so the carrier does not NAT the connection |
| Forward target on the ER605 | The Gateway's MetalLB address, not a node | Only that address has something listening on 80 and 443; a node's own address refuses the connection (see [NET-09-TRBL](../troubleshooting/NET-09-TRBL-Public-Port-Forwarding.md)) |
| Which Gateway | A separate `public` Gateway, never the LAN `homelab` Gateway | The LAN Gateway serves Grafana on `/` and ArgoCD; forwarding to it would publish them |
| Ports | TCP 80 and 443 only | 80 is needed for the Let's Encrypt HTTP-01 check and the redirect; 443 for the site |
| UPnP | Off on the ER605 (the ISP gateway has no UPnP setting) | Nothing on the LAN may open ports by itself |
| DMZ host | Disabled | A DMZ host would send *all* inbound traffic to one device and override the two rules |
| Remote administration | Off on the ISP gateway | It would put the gateway's own login page on the public address, and it competes for port 443 |

## Step 1 — Is the connection reachable at all?

Two checks before touching anything:

- **No carrier NAT.** The ISP gateway's Broadband Connection page shows a WAN address; `<PUBLIC_IP>` is that same address as seen from outside. If it were a private or `100.64.0.0/10` address, no forward could ever work.
- **The ISP gateway has a Port Forwarding page** (Security & Firewall). Some cellular home gateways do not.

The address is **dynamic**; keeping the DNS record current is handled in [SRV-25](../server/SRV-25-Public-Site-Operations.md).

## Step 2 — ISP gateway: reserve the ER605's address and forward

1. **Network Settings → IPv4 Address Distribution:** find the ER605 in the client list and reserve its address. The list shows no hostnames, so the ER605 is identified by its WAN address. Note it as `<ER605_WAN_IP>`. Without the reservation the forward would break the next time the ER605 gets a different address.
2. **Security & Firewall → Port Forwarding**, two rules:

| Name | Protocol | External port | Forward to | Port |
|---|---|---|---|---|
| web-http | TCP | 80 | `<ER605_WAN_IP>` | 80 |
| web-https | TCP | 443 | `<ER605_WAN_IP>` | 443 |

## Step 3 — ER605: forward to the Gateway

In the Omada controller (the Omada app or the web page; the ER605 is controller-managed, see [NET-04](./NET-04-Router-Configuration.md)): **Settings → Transmission → NAT → Virtual Servers**, two rules:

| Name | Interface | External port | Internal port | Internal IP | Protocol |
|---|---|---|---|---|---|
| public-http | WAN | 80 | 80 | `<PUBLIC_GW_IP>` | TCP |
| public-https | WAN | 443 | 443 | `<PUBLIC_GW_IP>` | TCP |

The internal IP is the **Gateway's** address (`kubectl get gateway -n envoy-gateway-system`), not a node address.

## Step 4 — Close the side doors

| Where | Check | Result |
|---|---|---|
| ISP gateway → System → Remote Administration | IPv4/IPv6 web access, ICMP echo and UDP traceroute all unchecked | Off |
| ISP gateway → Security & Firewall → DMZ Host | IPv4 and IPv6 | Disabled |
| ISP gateway → Network Settings | No UPnP entry in the menu | Not present |
| ER605 → UPnP | Switched off | Off |

## Step 5 — Test from outside

Before the real site existed, a throwaway page was served on the public Gateway: a one-pod nginx with a `HTTPRoute`, in a namespace labelled for the public Gateway, answering "reach-test OK" on both ports (the Gateway's 443 listener was plain HTTP for the test). From a phone with Wi-Fi **off**:

```
http://<PUBLIC_IP>/        → reach-test OK
http://<PUBLIC_IP>:443/    → reach-test OK
```

| Result | Meaning |
|---|---|
| Both pass | Ports are reachable; HTTP-01 certificates will work |
| 80 passes, 443 fails | A forward is wrong, or the ISP blocks 443 |
| 443 passes, 80 fails | Port 80 is blocked: use DNS-01 for certificates instead |
| Both time out | Forward chain broken or the ISP blocks inbound |
| Immediate "refused" | Something answered but nothing listens: the ER605 rule points at the wrong address ([NET-09-TRBL](../troubleshooting/NET-09-TRBL-Public-Port-Forwarding.md#connection-refused-from-outside)) |

Here both passed. The test namespace and page were deleted afterwards; the forwards and the Gateway stayed.

## Verification

From a phone on cellular data: `https://<DOMAIN>/` shows the site with a valid certificate. From the control plane, the router chain is also reachable from inside the house: `curl https://<DOMAIN>/api/health` returns `{"ok":true,...}`, which shows that this router passes traffic from the LAN side to its own public address (no hairpin problem), so a probe inside the cluster can test the whole public path ([SRV-25](../server/SRV-25-Public-Site-Operations.md)).

## Limits

- **The address is dynamic.** After a change, the site is down until the DNS record catches up (up to about 10 minutes with the updater of [SRV-25](../server/SRV-25-Public-Site-Operations.md)).
- **One uplink, one house.** A power or internet outage takes the site down, and the alerting runs inside the same house ([SRV-22](../server/SRV-22-Alerting-Reliability.md#limits)).
- **IPv4 only.** No IPv6 forward or DNS record exists.
- **Double forwarding must stay consistent.** If the ER605's address, the Gateway's MetalLB address or the ISP gateway's reservation changes, the matching rule has to change with it.
- **Inbound exposure is real.** Only two ports are open and both land on the public Gateway, but anything reachable there is attackable; see the limits and rate limits in [SRV-25](../server/SRV-25-Public-Site-Operations.md).
- **The ISP's policy can change.** Cellular home plans can start blocking inbound traffic or put the connection behind carrier NAT without notice.

## Files

| Item | Location | Contents |
|---|---|---|
| Port Forwarding rules | ISP gateway | TCP 80, 443 → `<ER605_WAN_IP>` |
| DHCP reservation | ISP gateway | ER605 → `<ER605_WAN_IP>` |
| Virtual Servers | ER605 / Omada controller | TCP 80, 443 → `<PUBLIC_GW_IP>` |

## Related

- [Internet Uplink](./NET-02-Internet-Uplink.md) — why the connection is double NAT
- [Router Configuration](./NET-04-Router-Configuration.md) — the ER605 and the Omada controller
- [Public Gateway and HTTPS](../server/SRV-23-Public-Gateway-and-HTTPS.md) — what the forwards lead to
- [NET-09-TRBL](../troubleshooting/NET-09-TRBL-Public-Port-Forwarding.md) — troubleshooting for this doc
- [Guide: Port Forwarding and a Changing Public IP](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/segmentation/Guide-Port-Forwarding-and-Dynamic-IP.md) — companion Guides repository
- [Guide: Double NAT and IP Passthrough](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/segmentation/Guide-Double-NAT-and-IP-Passthrough.md) — companion Guides repository
