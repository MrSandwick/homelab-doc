---
tags: [homelab-project, homelab, note, project, networking, radius, troubleshooting]
---

# Troubleshooting — FreeRADIUS Installation

> Companion to [FreeRADIUS Installation](../network/NET-05-FreeRADIUS-Installation.md), which records only the working configuration. This file records the problem hit during that work.

## `default_eap_type = md5`

**Relates to:** [EAP configuration](../network/NET-05-FreeRADIUS-Installation.md#eap-configuration).

> Background on what EAP is, and how PEAP/MSCHAPv2 relate to each other: [Guide-EAP-Methods](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-EAP-Methods.md).

**Symptom:** Windows refused to even prompt for credentials on the WPA2-Enterprise SSID — it failed instantly with *"Can't connect to this network,"* and nothing arrived at the RADIUS server at all (confirmed with `sudo freeradius -X` showing no `RADIUS:`-prefixed lines during a connection attempt, only the generic `AAA/BIND` / `AAA/AUTHEN/LOGIN` lines from method-list selection).

**Root cause:** `/etc/freeradius/3.0/mods-available/eap` had its **top-level** `default_eap_type` set to `md5`:

```
eap {
    default_eap_type = md5   # <- this line was the problem, not the ones below

    md5 {
        ...
    }
    peap {
        default_eap_type = mschapv2   # already correct — a different, nested setting
    }
    ttls {
        default_eap_type = mschapv2   # also unrelated — TTLS's own inner method
    }
}
```

EAP-MD5 is not offered by modern Windows or Android as a Wi-Fi authentication method, so the client abandoned the negotiation before a credentials prompt ever appeared — a different failure mode from a wrong password, and one that leaves no server-side log line pointing at the fix.

**Fix:** change only the top-level `default_eap_type` to `peap`; the nested `peap { default_eap_type = mschapv2 }` and `ttls { default_eap_type = mschapv2 }` blocks are settings for their own inner methods and were left untouched. Commands in [EAP configuration](../network/NET-05-FreeRADIUS-Installation.md#eap-configuration).

After the change, an Android device authenticated via WPA2-Enterprise/PEAP/MSCHAPv2 (Identity = MAC, Password = MAC), confirmed by a successful Wi-Fi connection and a matching `Access-Accept` line in `sudo freeradius -X` output.

## Related

- [FreeRADIUS Installation](../network/NET-05-FreeRADIUS-Installation.md) — the working configuration
- [Wireless / RADIUS Integration](../network/NET-06-Wireless-RADIUS-Integration.md)
- Guide: [EAP Methods](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-EAP-Methods.md)
