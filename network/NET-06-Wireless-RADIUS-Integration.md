---
tags: [homelab-project, homelab, note, project, networking, radius, wifi]
---

# Wireless / RADIUS Integration (Omada SSIDs)

> For a side-by-side comparison of what each mode below actually looks like from a connecting device, see [Guide-WiFi-Security-Modes](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-WiFi-Security-Modes.md).

> **Status update:** the WPA-Enterprise SSID documented below ("SuperAlga") was working and verified, but was wiped by an access-point factory reset during an unrelated Management VLAN experiment — see [Router Configuration](./NET-04-Router-Configuration.md). It needs to be recreated using the exact recipe in section 1 below; nothing about the recipe itself changed. The simpler WPA-Personal "Users" SSID survived/was trivially recreated, which is itself a useful data point — see the note at the end of the Router Configuration incident.

Three authentication approaches were evaluated on the EAP610 for the Admin SSID, in this order.

## 1. WPA-Enterprise — verified working

```
Security: WPA-Enterprise
RADIUS Profile: Authentication Server IP <SERVER_IP>, Port 1812, Password <ROUTER_RADIUS_SECRET>
VLAN: Custom -> 20
```

**Client behavior:** the device prompts once for Identity/Password (Identity = device MAC, Password = device MAC, per the `users` file in [FreeRADIUS Installation](./NET-05-FreeRADIUS-Installation.md)). Both Windows and Android were confirmed working after the `default_eap_type` fix documented there. Credentials are cached by the OS after the first successful connection, so in practice this is a one-time prompt per device, not a per-session one.

**Windows-specific note:** the OS also prompts to validate the RADIUS server's certificate on first connect ("Continue connecting?"). Since this deployment uses FreeRADIUS's default self-signed certificate rather than a CA-issued one, this prompt is expected and was accepted manually. A production-grade deployment would replace it with a real certificate to remove this step and the associated trust-on-first-use risk.

## 2. PPSK with RADIUS — evaluated, not adopted

Omada also offers **PPSK (Private Pre-Shared Key) with RADIUS**: the RADIUS server, keyed by the client's MAC address, tells the controller which password that specific device should be prompted for. This still requires the client to type a password (the RADIUS-supplied one) — it is **not** password-free, just per-device rather than shared. Testing showed the same MAC-as-password behavior as plain WPA-Enterprise, so it wasn't adopted as a separate mechanism; WPA-Enterprise was simpler to reason about for the same practical result.

## 3. MAC-Based Authentication with Empty Password — configured, not yet fully verified

Found under `Settings → Wireless Networks → MAC-Based Authentication`, this is Omada's dedicated feature for fully silent MAC filtering via RADIUS — no credential prompt at all:

```
MAC-Based Authentication: Enable
Type: RADIUS Auth
SSID: <target SSID>          # requires Security = None or WPA-Personal on that SSID,
                              # not WPA-Enterprise
RADIUS Profile: <same profile as above>
MAC Address Format: aa:bb:cc:dd:ee:ff   # colon-separated — differs from the format below
Empty Password: Enable
```

**Open item:** the `users` file on the RADIUS server currently stores MAC addresses without separators (`aabbccddeeff`), matching the format the WPA-Enterprise flow sends. This feature's `MAC Address Format` field defaults to colon-separated (`aa:bb:cc:dd:ee:ff`) instead. Before this mode can work end-to-end, either:

- the `MAC Address Format` dropdown needs to be changed to a no-separator option, if one is available, or
- the `users` file entries need to be duplicated in colon-separated form so both formats resolve correctly

This mismatch was identified but not yet resolved at time of writing — tracked in the status table in [Project Overview](./NET-00-Project-Overview.md).

## Users SSID (VLAN 10)

Kept intentionally simple: WPA-Personal with a normal shared passphrase, no RADIUS involvement, `VLAN: Custom → 10`. This SSID is for everyday household devices and doesn't need per-device filtering.

## Related

- [FreeRADIUS Installation](./NET-05-FreeRADIUS-Installation.md)
- [Router Configuration](./NET-04-Router-Configuration.md) — the controller-connectivity incident triggered while editing these SSIDs
- Guides: [Wi-Fi Security Modes](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-WiFi-Security-Modes.md) · [EAP Methods](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-EAP-Methods.md) · [MAC Address Filtering and Spoofing](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-MAC-Address-Filtering-and-Spoofing.md)
