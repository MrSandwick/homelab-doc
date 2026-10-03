---
tags: [homelab-project, homelab, note, project, networking, monitoring]
---

# Traffic Monitoring

> Status: 🟢 verified working end-to-end.

Problems hit during this work are recorded separately in [NET-08-TRBL](./NET-08-TRBL-Monitoring.md).

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

> What port mirroring and promiscuous mode actually mean, and why a network card needs to be told to stop filtering: [Guide-Port-Mirroring-and-Promiscuous-Mode](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Port-Mirroring-and-Promiscuous-Mode.md).

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

### NIC offloading disabled on the mirror interface

Hardware offloading (TSO/GSO/GRO) merges several real packets into one before software sees them, which breaks per-packet accuracy on a monitoring interface. Disabled on `enp2s0`:

```
sudo ethtool -K enp2s0 gro off gso off tso off
```

Background: [NET-08-TRBL](./NET-08-TRBL-Monitoring.md#nic-offloading-corrupts-monitored-packet-sizes).

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
      - -m
      - "<MGMT_SUBNET>/24,<USERS_SUBNET>/24,<ADMIN_SUBNET>/24"
```

```
docker compose up -d
docker compose ps
```

- `network_mode: host` on both containers — required so ntopng can see the physical `enp2s0` interface directly (Docker's default bridge network would hide it), and so it can reach Redis on `127.0.0.1:6379` within the same network namespace.
- `./data:/var/lib/ntopng` — persists ntopng's host/alert database across container restarts.
- `-m` — declares the local subnets. The mirror interface has no IP of its own, so ntopng cannot infer them; without it every observed subnet raises a `Ghost Networks` alert ([NET-08-TRBL](./NET-08-TRBL-Monitoring.md#ghost-networks-alerts)).
- Image tag `:latest` — the official image is not published under `:stable` ([NET-08-TRBL](./NET-08-TRBL-Monitoring.md#ntopntopngstable-tag-does-not-exist)).

Dashboard: `http://<SERVER_IP>:3000`, default login `admin` / `admin` — change on first login.

### Using ntopng's flow inspector to evaluate an alert

Worth documenting as a repeatable workflow: ntopng's per-flow detail view (click any flow in `Flows`) is useful for triaging a flagged alert rather than reacting to the score alone. Example encountered: a flow from an Admin-VLAN host to an Akamai CDN IP, classified as `HTTP.Microsoft365` over plain HTTP (port 80), triggered a `Mismatching protocol with IP address` alert at the maximum score (100), tagged with a MITRE ATT&CK ID.

Checking the ID against the real MITRE ATT&CK framework showed it corresponds to an unrelated technique (`Rogue Domain Controller`) that has nothing to do with a plain CDN-hosted HTTP request — a reminder that ntopng Community's automatic MITRE tagging for nDPI risk flags can be a loose, approximate mapping rather than a precise classification, and shouldn't be taken at face value without checking what the underlying nDPI risk actually detected. The likely explanation for the alert itself: Microsoft serves a meaningful share of Microsoft 365 traffic through third-party CDNs like Akamai, so nDPI's classifier (which associates certain protocols with known IP ranges) flagged a mismatch between "looks like Microsoft365" and "IP belongs to Akamai, not Microsoft" — a common false-positive pattern for any vendor that uses a CDN. Treated as benign after confirming no matching suspicious process was running on the source host.

## Related

- [NET-08-TRBL](./NET-08-TRBL-Monitoring.md) — troubleshooting for this doc
- [Project Overview](./NET-00-Project-Overview.md)
- Server docs: [Docker Installation](../server/SRV-04-Docker-Installation.md) — Docker installation details
- Guide: [Port Mirroring and Promiscuous Mode](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Port-Mirroring-and-Promiscuous-Mode.md)
