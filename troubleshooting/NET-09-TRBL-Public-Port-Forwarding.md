---
tags: [homelab, project, networking, port-forwarding, double-nat, troubleshooting]
---

# Troubleshooting — Public Port Forwarding

> Companion to [Public Port Forwarding](../network/NET-09-Public-Port-Forwarding.md), which records only the working configuration. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [The ER605 could not be found in the ISP gateway's client list](#the-er605-could-not-be-found-in-the-isp-gateways-client-list) | [Step 2](../network/NET-09-Public-Port-Forwarding.md#step-2--isp-gateway-reserve-the-er605s-address-and-forward) |
| [The Omada controller was not known to be anywhere](#the-omada-controller-was-not-known-to-be-anywhere) | [Step 3](../network/NET-09-Public-Port-Forwarding.md#step-3--er605-forward-to-the-gateway) |
| [Connection refused from outside](#connection-refused-from-outside) | [Step 5](../network/NET-09-Public-Port-Forwarding.md#step-5--test-from-outside) |

## The ER605 could not be found in the ISP gateway's client list

**Symptom:** the ISP gateway's client list showed several `192.168.1.x` entries and no hostnames, so the ER605 could not be told apart.

**Root cause:** the gateway lists clients by address and MAC only; the ER605 does not announce a name.

**Fix:** the ER605 was identified by its WAN address and that address was reserved. The reservation matters as much as finding it: a forward to an address that can change is a time bomb.

## The Omada controller was not known to be anywhere

**Symptom:** the ER605 is adopted, so its settings are owned by the controller, but nobody remembered where the controller runs. A check of the Windows PC for a local controller (ports, services, install folders, containers) found nothing.

**Root cause:** the controller is reached through the **Omada app**; there is no controller software on the PC or the server that anyone had opened by hand.

**Fix:** the Virtual Server rules were added in the app. Worth noting for [NET-04](../network/NET-04-Router-Configuration.md): the ER605 is managed through the controller, so its settings belong there; a change made in the router's own web page is not the place to rely on.

## Connection refused from outside

**Symptom:** from a phone on cellular data, `http://<PUBLIC_IP>/` failed immediately with "refused to connect", not after a timeout. The Envoy access log of the public Gateway (`kubectl -n envoy-gateway-system logs -l gateway.envoyproxy.io/owning-gateway-name=public -f`) showed only the earlier local `curl` tests and no request from the phone.

**What was found:** a quick refusal means a device on the path answered with a reset, which a silent drop never does. Combined with the empty Envoy log, the request had left the ISP gateway and reached something that had nothing listening on that port. The Virtual Server rules on the ER605 had the **worker node's** own address as the internal IP.

**Root cause:** a node has nothing on ports 80 and 443. The Gateway's Service is reachable on the MetalLB address, where the Envoy pod's ports are announced; a node address only exposes the high NodePorts.

**Fix:** both rules were changed to the Gateway's MetalLB address `<PUBLIC_GW_IP>`. The next test returned the test page on both ports.

**Check added to the procedure:** follow the Envoy access log while testing from outside. Requests in the log mean the routers work and the problem is in the cluster; nothing in the log means the problem is in the forward chain, and the rules are checked from the outside in (ISP gateway → ER605 → Gateway address).

## Related

- [Public Port Forwarding](../network/NET-09-Public-Port-Forwarding.md) — the working configuration
- [NET-04-TRBL](./NET-04-TRBL-Router-Configuration.md) — earlier controller incidents
- [Guide: Port Forwarding and a Changing Public IP](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/segmentation/Guide-Port-Forwarding-and-Dynamic-IP.md) — companion Guides repository
