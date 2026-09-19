---
tags: [homelab-project, homelab, note, project, networking, radius]
---

# FreeRADIUS Installation

> New to RADIUS/AAA, or the "client" and "shared secret" terminology below? See [Guide-RADIUS-and-AAA](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-RADIUS-and-AAA.md).

Installed on the same server documented in the [server section](../server/SRV-README.md) (Ubuntu Server, systemd). This doc covers only the RADIUS-specific setup — OS and Docker installation live in the [server section](../server/SRV-README.md).

## Install

```
sudo apt update
sudo apt install freeradius freeradius-utils -y
sudo systemctl enable freeradius
sudo systemctl status freeradius
```

## Client (NAS) configuration

Each RADIUS client (router, access point) gets its own entry in `clients.conf`, each with its **own** shared secret — reusing a single secret across clients was deliberately avoided so a leaked config for one device doesn't expose the others:

```
sudo nano /etc/freeradius/3.0/clients.conf
```

```
client router {
    ipaddr = <ADMIN_GATEWAY_IP>
    secret = <ROUTER_RADIUS_SECRET>
}

client eap610 {
    ipaddr = <AP_MGMT_IP>
    secret = <AP_RADIUS_SECRET>
}
```

Secrets were generated with:

```
openssl rand -base64 24
```

## MAC-address user entries

Each device permitted onto the Admin Wi-Fi network is a RADIUS "user" whose username *and* password are its own MAC address, lowercase with no separators (`aabbccddeeff`) — matching the MAC address format the access point's RADIUS client sends for the WPA-Enterprise flow described in [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md):

```
sudo nano /etc/freeradius/3.0/users
```

```
<ADMIN_LAPTOP_MAC> Cleartext-Password := "<ADMIN_LAPTOP_MAC>"
<ADMIN_PHONE_MAC>  Cleartext-Password := "<ADMIN_PHONE_MAC>"
```

> Using the MAC address itself as the password is weak by design — MAC addresses are visible in plaintext 802.11 management frames and are trivially spoofable with off-the-shelf tools. It was a deliberate choice for this project's threat model (a home network, where the goal is *filtering by known device* rather than defending against a determined local attacker), not a general security recommendation. See [Guide-MAC-Address-Filtering-and-Spoofing](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-MAC-Address-Filtering-and-Spoofing.md) and "Known limitations" below.

## Verifying the daemon locally

```
sudo systemctl restart freeradius
radtest <TEST_USER> <TEST_PASSWORD> localhost 0 <LOCALHOST_CLIENT_SECRET>
```

A successful response includes `Access-Accept`. The `localhost` secret comes from the `client localhost { }` block that ships in `clients.conf` by default (`testing123` unless changed) — this is a separate entry from the `router`/`eap610` clients configured above, since `radtest` run on the server itself talks to the `127.0.0.1` client entry, not to either real device's entry.

## Real troubleshooting: `default_eap_type = md5`

> Background on what EAP even is, and how PEAP/MSCHAPv2 relate to each other: [Guide-EAP-Methods](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-EAP-Methods.md).

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

EAP-MD5 isn't offered by modern Windows or Android as a Wi-Fi authentication method at all, so the client abandoned the negotiation before a credentials prompt ever appeared — an entirely different failure mode from a wrong password, and one that leaves no server-side log line pointing directly at the fix.

**Fix:** change only the top-level `default_eap_type` to `peap`; the nested `peap { default_eap_type = mschapv2 }` and `ttls { default_eap_type = mschapv2 }` blocks are unrelated settings for their own respective inner methods and were left untouched:

```
sudo nano /etc/freeradius/3.0/mods-available/eap
# default_eap_type = peap
sudo systemctl restart freeradius
```

After this change, an Android device authenticated successfully via WPA2-Enterprise/PEAP/MSCHAPv2 (Identity = MAC, Password = MAC), confirmed both by a successful Wi-Fi connection and by a matching `Access-Accept` line in `sudo freeradius -X` output.

## Debugging workflow used throughout

```
sudo systemctl stop freeradius
sudo freeradius -X
# attempt the Wi-Fi connection from a client device in parallel, watch this terminal
# ...
sudo systemctl start freeradius   # return to normal background operation when done
```

## Known limitations / accepted trade-offs

- MAC-as-password is weak (see note above); acceptable for this project's threat model, not recommended as a general practice
- ~~`users` file passwords were briefly set to a placeholder value (`"1"`) mid-troubleshooting~~ — **resolved:** replaced with real, non-trivial passwords once WPA-Enterprise authentication was confirmed working end-to-end
- `Require Message-Authenticator` was disabled on the RADIUS profile after it appeared to interfere with the MAC-based authentication flows under test; the security implication of disabling it was not fully investigated and should be revisited

## Related

- [Wireless / RADIUS Integration](./NET-06-Wireless-RADIUS-Integration.md)
- Guides: [RADIUS and AAA](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-RADIUS-and-AAA.md) · [EAP Methods](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-EAP-Methods.md) · [MAC Address Filtering and Spoofing](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-MAC-Address-Filtering-and-Spoofing.md)
