---
tags: [homelab-project, homelab, note, project, networking, security, troubleshooting]
---

# Troubleshooting — Access Control and Isolation

> Companion to [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md), which records only the working configuration. This file records the problems hit during that work.

## Two Gateway ACL rule-authoring mistakes

**Relates to:** [Management ACLs](./NET-07-Access-Control-and-Isolation.md#whats-implemented-on-the-real-network-management-acls). Both were caught before they caused a lockout or a leak.

1. **Action/name mismatch.** The first version of the Permit rule was named `Permit-to-Management` but had its **Action** field still set to `Deny` — which would have blocked the very devices it was meant to allow. Caught by re-reading the rule table rather than assuming the name matched the behavior; fixed by setting Action to `Permit`.
2. **An overly broad rule silently defeated the narrow one.** A second rule, `Allow-Admin-to-Network` (Source: `Network:Admin` — the *entire* Admin VLAN, not just the trusted IP group; Destination: `Default, Users`; Action: `Permit`), had been created earlier during testing and was still active, positioned **above** the intended Deny rule. Since it matched first, it granted management access to every device on the Admin VLAN, making the narrower Permit/Deny pair irrelevant. Found by reviewing the full rule list end-to-end rather than testing only the two rules just written; removed once identified.

## Related

- [Access Control and Isolation](./NET-07-Access-Control-and-Isolation.md) — the working configuration
- [NET-04-TRBL](./NET-04-TRBL-Router-Configuration.md) — the controller-management-port issue still open
- Guide: [Network Isolation vs. ACLs](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Network-Isolation-vs-ACLs.md)
