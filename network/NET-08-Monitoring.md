---
tags: [homelab-project, homelab, note, project, networking, monitoring]
---

# Traffic Monitoring

> Status: 🟢 verified working — ntopng dashboard showing live traffic across all VLANs.

## Goal

Basic visibility into per-device traffic on the network, as a step toward the longer-term goal of adding IDS/IPS (Suricata/Zeek) analysis with LLM-assisted alert triage — not started yet, tracked separately in [Project Overview](./NET-00-Project-Overview.md).

## Server-local monitoring (vnstat / iftop)

For traffic actually passing through the server's own interface:

```
sudo apt install vnstat iftop -y
sudo systemctl enable --now vnstat

vnstat -l -i eno1       # live throughput
vnstat -d -i eno1       # daily totals
sudo iftop -i eno1      # live per-connection breakdown
```

These only see traffic that actually traverses the server's own NIC — not general inter-device traffic elsewhere on the network. For that, port mirroring on the switch is needed (below).

## Network-wide visibility: switch port mirroring

> What port mirroring and promiscuous mode actually mean, and why a network card needs to be told to stop filtering: [Guide-Port-Mirroring-and-Promiscuous-Mode](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-Port-Mirroring-and-Promiscuous-Mode.md).

The server has a second, otherwise-unused NIC (`enp2s0`) free for exactly this purpose. Configured via the switch's Easy Smart Utility (see the web-UI limitation noted in [Hardware Selection](./NET-01-Hardware-Selection.md)):

```
Monitoring -> Port Mirror
Port Mirror Status: Enable
Mirroring Port: 2
Mirrored Port 1 (router uplink trunk): Ingress = Enable, Egress = Enable
```

**Note:** the mirroring port was originally set to Port 5, then moved to **Port 2** once Port 5 was needed for the newly added Kubernetes worker node (see [Second Node Setup](../server/SRV-06-Second-Node-Setup.md)) — see [VLAN Design and Switch Configuration](./NET-03-VLAN-Design-and-Switch-Configuration.md) for the current full port map. Moving a mirroring port is safe as long as the switch's Mirroring Port setting and the physical cable are updated together — mismatching them silently breaks monitoring without affecting the actual network traffic at all.

This copies all traffic crossing the router-uplink trunk port — i.e. everything moving between VLANs and out to the internet — onto the mirroring port, which is physically wired to the server's second NIC.

On the server side, that interface carries no IP of its own; it only needs to be in promiscuous mode to receive traffic that isn't addressed to it:

```
sudo ip link set enp2s0 promisc on
ip addr show enp2s0   # confirm PROMISC appears in the interface flags
```

### Real troubleshooting: NIC offloading corrupts monitored packet sizes

ntopng's own startup log flagged a real accuracy problem on the mirrored interface:

```
Packets exceeding the expected max size have been received [enp2s0][len: 1646][max len: 1518]
WARNING: If TSO/GRO is enabled, please disable it for best accuracy
```

**Root cause:** hardware offloading features (TSO/GSO on transmit, GRO on receive) merge multiple real packets into one larger "virtual" packet before software ever sees them — a performance optimization that's actively harmful for a monitoring interface, since ntopng needs to see genuine per-packet boundaries and timing, not merged ones. A merged "packet" of 1646 bytes is larger than Ethernet's real 1518-byte maximum, which is what tipped this off.

**Fix:**

```
sudo ethtool -K enp2s0 gro off gso off tso off
```

**Made persistent** (ethtool settings don't survive a reboot) via a small systemd unit:

```
sudo nano /etc/systemd/system/disable-offload-enp2s0.service
```

```ini
[Unit]
Description=Disable NIC offloading on the mirrored monitoring interface
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -K enp2s0 gro off gso off tso off

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now disable-offload-enp2s0.service
```

## ntopng (Docker, with Redis)

The official `ntop/ntopng` image needs a Redis instance for full functionality (alerts, timeseries history) and benefits from a persistent data volume — the single-container approach was replaced with a small `docker-compose.yml`:

```yaml
services:
  ntopng-redis:
    image: redis:alpine
    container_name: ntopng-redis
    restart: unless-stopped
    network_mode: host

  ntopng:
    image: ntop/ntopng:latest
    container_name: ntopng
    restart: unless-stopped
    network_mode: host
    depends_on:
      - ntopng-redis
    volumes:
      - ./data:/var/lib/ntopng
    command:
      - --community
      - -i
      - enp2s0
      - -r
      - 127.0.0.1:6379
      - -w
      - "3000"
```

```
docker compose up -d
docker compose ps
```

- `network_mode: host` on both containers — required so ntopng can see the physical `enp2s0` interface directly (Docker's default bridge network would hide it), and so it can reach Redis on `127.0.0.1:6379` within the same network namespace.
- `./data:/var/lib/ntopng` — persists ntopng's host/alert database across container restarts.

**Real troubleshooting: `ntop/ntopng:stable` tag doesn't exist.** The first attempt used `image: ntop/ntopng:stable`, which failed with `failed to resolve reference ... not found` — the official image is currently only published under the `:latest` tag on Docker Hub. Switched to `ntop/ntopng:latest`.

Dashboard: `http://<SERVER_IP>:3000`, default login `admin` / `admin` — change on first login.

### Real troubleshooting: "Ghost Networks" alerts

**Symptom:** two persistent alerts, `Ghost Networks`, for both the management subnet and the Admin subnet, plus a banner reading `No local hosts detected, although there are active hosts. Please make sure local networks have been properly configured (-m parameter).`

**Root cause:** the monitored interface (`enp2s0`) has no IP address of its own (by design — it's a passive mirror), so ntopng has no way to automatically infer which subnets should be considered "local" the way it normally would from an interface's own address. Every subnet it observes traffic from — correctly, since the whole point of mirroring is to see traffic from multiple VLANs at once — gets flagged as unexpected.

**Fix (identified, not yet fully applied at time of writing):** explicitly declare the known local subnets, either via the `-m` startup parameter (e.g. `-m "<MGMT_SUBNET>/24,<USERS_SUBNET>/24,<ADMIN_SUBNET>/24"` added to the `command:` block above) or via `Settings → Networks` in the UI. Tracked as a follow-up; the alerts are cosmetic (they don't block monitoring) but worth clearing for a clean dashboard.

### Using ntopng's flow inspector to evaluate an alert

Worth documenting as a repeatable workflow: ntopng's per-flow detail view (click any flow in `Flows`) is useful for triaging a flagged alert rather than reacting to the score alone. Example encountered: a flow from an Admin-VLAN host to an Akamai CDN IP, classified as `HTTP.Microsoft365` over plain HTTP (port 80), triggered a `Mismatching protocol with IP address` alert at the maximum score (100), tagged with a MITRE ATT&CK ID.

Checking the ID against the real MITRE ATT&CK framework showed it corresponds to an unrelated technique (`Rogue Domain Controller`) that has nothing to do with a plain CDN-hosted HTTP request — a reminder that ntopng Community's automatic MITRE tagging for nDPI risk flags can be a loose, approximate mapping rather than a precise classification, and shouldn't be taken at face value without checking what the underlying nDPI risk actually detected. The likely explanation for the alert itself: Microsoft serves a meaningful share of Microsoft 365 traffic through third-party CDNs like Akamai, so nDPI's classifier (which associates certain protocols with known IP ranges) flagged a mismatch between "looks like Microsoft365" and "IP belongs to Akamai, not Microsoft" — a common false-positive pattern for any vendor that uses a CDN. Treated as benign after confirming no matching suspicious process was running on the source host.

## Related

- [Project Overview](./NET-00-Project-Overview.md)
- Server docs: [Docker Installation](../server/SRV-04-Docker-Installation.md) — Docker installation details
- Guide: [Port Mirroring and Promiscuous Mode](<LINK_TO_NETWORK_GUIDES_REPO>/Guide-Port-Mirroring-and-Promiscuous-Mode.md)
