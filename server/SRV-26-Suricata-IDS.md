---
tags: [homelab, project, security, ids, suricata, monitoring]
---

# Suricata IDS on the Mirror Port

> Status: 🟢 **Capturing, alerting, logging and rotating.** 🟡 Several checks are still open — see [Open checks](#open-checks). Alert triage is covered in [AI Alert Triage](./SRV-27-AI-Alert-Triage.md).

Runs on the control-plane node (`<HOSTNAME>`), capture NIC `enp2s0` — the same mirror NIC ntopng reads, see [Traffic Monitoring](../network/NET-08-Monitoring.md).

Problems hit during this work are recorded separately in [SRV-26-TRBL](../troubleshooting/SRV-26-TRBL-Suricata-IDS.md).

## Contents

1. [Goal](#goal)
2. [Data path](#data-path)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Prepare the capture NIC](#step-1--prepare-the-capture-nic)
5. [Step 2 — Suricata container](#step-2--suricata-container)
6. [Step 3 — BPF capture filter](#step-3--bpf-capture-filter)
7. [Step 4 — Rules](#step-4--rules)
8. [Step 5 — Logging](#step-5--logging)
9. [Step 6 — Log rotation](#step-6--log-rotation)
10. [Step 7 — Test alert](#step-7--test-alert)
11. [Step 8 — Health checks](#step-8--health-checks)
12. [What the sensor cannot see](#what-the-sensor-cannot-see)
13. [First weeks: what the alerts looked like](#first-weeks-what-the-alerts-looked-like)
14. [Triage log (reviewed alerts)](#triage-log-reviewed-alerts)
15. [Open checks](#open-checks)

## Goal

Passive intrusion *detection* for the whole home network: see what the segmented network actually does, flag known-bad patterns, and keep a searchable record. This is an **IDS, not an IPS**: the sensor receives a mirrored copy of the traffic, so it can raise alerts but cannot block anything. Blocking stays the router's job ([Router Configuration](../network/NET-04-Router-Configuration.md)).

## Data path

```
Internet ── 5G gateway ── router ══ trunk ══ switch ── APs / servers / PCs
                                              │
                    port mirror (ingress + egress of the router-uplink port)
                                              ▼
                 server NIC enp2s0 (no IP, promiscuous, offloads off)
                                              ▼
                Suricata (Docker, host network, AF_PACKET capture)
                                              ▼
            /home/<USERNAME>/suricata/log/eve.json ──► logrotate (daily, 14 copies)
                                              ▼
                     aggregator + local LLM (see SRV-27)
```

Because the mirror covers the router-uplink port, the sensor sees traffic that crosses VLANs or leaves for the internet. See [What the sensor cannot see](#what-the-sensor-cannot-see).

## Prerequisites

- Port mirror configured on the switch as described in [Traffic Monitoring](../network/NET-08-Monitoring.md), with the mirror destination cabled to the server's second NIC.
- Docker with the Compose plugin ([Docker Installation](./SRV-04-Docker-Installation.md)).
- The `acl` package (`sudo apt install acl`), needed for log rotation below.

## Step 1 — Prepare the capture NIC

Two oneshot systemd units put the NIC into the state Suricata needs after every boot. The offload unit already existed for ntopng ([Traffic Monitoring](../network/NET-08-Monitoring.md)); the promiscuous-mode unit was added here so the setting survives a reboot instead of being applied by hand.

`/etc/systemd/system/promisc-enp2s0.service`
```ini
[Unit]
Description=Enable promiscuous mode on enp2s0
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/ip link set enp2s0 promisc on

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/disable-offload-enp2s0.service`
```ini
[Unit]
Description=Disable GRO/GSO/TSO on enp2s0
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/ethtool -K enp2s0 gro off gso off tso off

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now promisc-enp2s0.service disable-offload-enp2s0.service
```

- **Promiscuous mode:** a NIC normally drops frames not addressed to it. Mirrored traffic is addressed to *other* hosts, so without this flag almost nothing would reach the sensor ([Guide: Port Mirroring and Promiscuous Mode](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/operations/Guide-Port-Mirroring-and-Promiscuous-Mode.md)).
- **Offloads off (GRO/GSO/TSO):** these features merge or split packets inside the NIC/driver. A sensor then sees frames that never existed on the wire, which breaks flow reassembly and produces bogus decoder alerts.

Verify:
```
ip link show enp2s0 | grep -o PROMISC
ethtool -k enp2s0 | grep -E 'generic-receive-offload|generic-segmentation-offload|tcp-segmentation-offload'
```

> 🟡 The unit files above are a reconstructed minimal form. Replace them with the real text from `systemctl cat promisc-enp2s0 disable-offload-enp2s0` if the two differ.

## Step 2 — Suricata container

`/home/<USERNAME>/suricata/docker-compose.yml`
```yaml
services:
  suricata:
    image: jasonish/suricata:latest
    container_name: suricata
    restart: unless-stopped
    network_mode: host
    cap_add: [NET_ADMIN, NET_RAW, SYS_NICE]
    mem_limit: 2g
    command: ["-i", "enp2s0", "not (ether proto 0x8899 or ether proto 0x890d or ether proto 0x88cc)"]
    volumes:
      - ./etc:/etc/suricata
      - ./lib:/var/lib/suricata
      - ./log:/var/log/suricata
```

| Setting | Why |
|---|---|
| `image: jasonish/suricata` | Community image (Suricata 8.0.x at time of writing) that ships the Emerging Threats Open ruleset |
| `network_mode: host` | Capture on the physical NIC; a bridged container network would hide `enp2s0` entirely |
| `cap_add: NET_ADMIN, NET_RAW, SYS_NICE` | Raw packet capture, interface control, thread priorities |
| `mem_limit: 2g` | Keeps the sensor from starving Kubernetes workloads on the same node |
| `-i enp2s0` | AF_PACKET capture on the mirror NIC |
| BPF expression (last argument) | Discards link-layer management frames before Suricata decodes them (next section) |
| `./etc`, `./lib`, `./log` | Config, rules and logs live on the host and survive container re-creation |

```
cd ~/suricata && docker compose up -d
docker logs --tail 20 suricata
```

`docker compose up -d` reports success even if the container crashes a second later, so always confirm the state rather than trusting the exit code:

```
docker inspect -f '{{.State.Status}} restarts={{.RestartCount}}' suricata   # expect: running restarts=0
```

## Step 3 — BPF capture filter

LLDP (`0x88cc`) and two other link-layer management protocols (`0x8899`, `0x890d`) from the network gear were arriving on the mirror and producing a constant stream of "unknown Ethertype" decoder alerts. They carry no IP traffic, so they are removed at capture time.

Validate an expression before putting it in the compose file — `tcpdump -d` prints BPF bytecode only if the expression parses:

```
sudo tcpdump -d -i enp2s0 'not (ether proto 0x8899 or ether proto 0x890d or ether proto 0x88cc)'
```

A filter that drops its `not` or its parentheses silently rejects *every* packet and puts the container into a restart loop — see [SRV-26-TRBL](../troubleshooting/SRV-26-TRBL-Suricata-IDS.md#bpf-filter-rejecting-every-packet).

## Step 4 — Rules

The Emerging Threats **Open** ruleset was loaded: 53,203 rules at the time of writing. Inspecting `lib/rules/suricata.rules` as a normal user returns `Permission denied`, because the files belong to the container user — use `sudo`. Rule update scheduling is not decided yet (see [Open checks](#open-checks)).

## Step 5 — Logging

`eve.json` holds one JSON object per line. Alert events carry protocol metadata (`dns`, `http`, `tls.sni`, `proto`, `app_proto`); separate `http` events (with response codes) and periodic `stats` events are written to the same file. `fast.log` and `stats.log` are written as well.

`stats: interval: 24` is set in `etc/suricata.yaml` (default 8 s) to cut the volume of `stats` events. The change applies only after `docker compose restart suricata`.

## Step 6 — Log rotation

The container writes logs as a host account named `systemd-network` (same numeric uid), so logrotate must rotate as that user, and that user must be able to traverse the home directory.

`/etc/logrotate.d/suricata-docker`
```
/home/<USERNAME>/suricata/log/eve.json /home/<USERNAME>/suricata/log/stats.log /home/<USERNAME>/suricata/log/fast.log {
    su systemd-network systemd-network
    daily
    rotate 14
    compress
    delaycompress
    dateext
    missingok
    notifempty
    copytruncate
}
```

| Directive | Why |
|---|---|
| `su systemd-network systemd-network` | Rotate as the owner of the files; plain root rotation inside a user home directory fails, see [SRV-26-TRBL](../troubleshooting/SRV-26-TRBL-Suricata-IDS.md#logrotate-permission-denied-in-a-user-home-directory) |
| `copytruncate` | Suricata keeps the file open and there is no reload hook, so copy and truncate in place |
| `delaycompress` | Leave the newest rotated copy uncompressed so tools can still read it |
| `dateext`, `rotate 14` | Date-stamped names, two weeks of history |

```
sudo setfacl -m u:systemd-network:x /home/<USERNAME>   # execute-only: lets the log owner traverse the home directory
sudo logrotate -f /etc/logrotate.d/suricata-docker      # force one rotation to test
ls -l ~/suricata/log && systemctl list-timers logrotate.timer
```

## Step 7 — Test alert

From any LAN client:
```
curl http://testmyids.com
```
Expected alert in `eve.json`: `GPL ATTACK_RESPONSE id check returned root`.
```
grep -c 'id check returned root' ~/suricata/log/eve.json
```
(Typing `testmynids.org` instead fails to resolve and generates nothing.)

## Step 8 — Health checks

```
# capture loss (fraction of kernel packets dropped); about 0.036% at time of writing
jq -r 'select(.event_type=="stats") | .stats.capture | "\(.kernel_packets) \(.kernel_drops)"' ~/suricata/log/eve.json | tail -1

# stats interval (consecutive timestamps should be ~24 s apart)
jq -r 'select(.event_type=="stats") | .timestamp' ~/suricata/log/eve.json | tail -5
```

## What the sensor cannot see

- **Encrypted content.** For TLS only the handshake metadata (SNI, certificate) is visible, never the payload. The same applies to HTTPS on any public service: once a site is served over HTTPS, response codes disappear from the IDS and the web server's own access log becomes the source of truth.
- **DNS over HTTPS.** Devices that resolve names through DoH bypass the router's DNS, so those lookups never appear as DNS events. The ET DoH signatures flag some of it.
- **Traffic inside a single VLAN.** Frames that never cross the router-uplink port are never mirrored.
- **Partial flows.** A mirror port can miss packets or start mid-connection, which produces `SURICATA …` stream/app-layer anomaly alerts. These are treated as sensor noise, not as findings.

## First weeks: what the alerts looked like

About 92,500 alerts in 15 days, of which roughly 99% belonged to a few known-harmless families:

| Family | Why it fires |
|---|---|
| `SURICATA …` stream / app-layer anomalies | Mirror port and partial flows (sensor artefact) |
| STUN | WebRTC / game / chat clients |
| Discord, OpenAI, Spotify, Steam, APT, GitHub release assets, Telegram domains | Informational "observed application" signatures |
| Microsoft / Android connectivity checks | OS network-probe requests |
| DNS over HTTPS | Browser/OS DoH, informational |

The remaining few dozen signatures were reviewed by hand; the interesting ones are below. The automated version of this process is described in [AI Alert Triage](./SRV-27-AI-Alert-Triage.md).

## Triage log (reviewed alerts)

| Alert | What it turned out to be | Assessment |
|---|---|---|
| `ET DROP Spamhaus DROP Listed Traffic Inbound` (6) | ICMP from four addresses in an ISP backbone block (WHOIS: the home internet provider's business network) to two LAN PCs | Benign ISP-side ICMP. Why that block is on the list is unexplained |
| `GPL ATTACK_RESPONSE id check returned root` (2) | The deliberate test in step 7 | Expected |
| `ET EXPLOIT Possible OpenSSL HeartBleed Large HeartBeat Response` (1) | Coincided with a Telegram client talking to Telegram datacentres on port 443 (MTProto) | Consistent with a known false positive; not independently confirmed |
| `ET HUNTING … Dotted Quad Host MZ Response`, `ET INFO Packed Executable Download` (82) | User-Agent `Microsoft-Delivery-Optimization` | Windows Update. The User-Agent can be forged, so the noise rule is restricted to exactly these two signatures |
| `ET ADWARE_PUP Antibody Software Installed`, `ET USER_AGENTS Suspicious User-Agent (Embarcadero URI Client)` (1 each) | Same device, same minute, looks like an installer/updater check | 🟡 Probably a utility install; unconfirmed |
| `ET INFO Vulnerable Java Version 1.8.x` (18) | Fires on any Java 1.8.x | Hygiene item: update Java |
| TLD heuristics (`.cc`, `.top`, `.su`, `.biz`, `.world`, `.life`, `.fit`) | Streaming, ad-tech and adult sites | Informational; no command-and-control indicators |
| `ET INFO DNS Query for TOR Hidden Domain` (101) | Always the same probe name; the router answers `.onion` locally with NXDOMAIN | No leak |

## Open checks

- [ ] 🟡 Replace the two unit files above with the real output of `systemctl cat promisc-enp2s0 disable-offload-enp2s0` if they differ
- [ ] 🟡 `stats` events are ~24 s apart after `docker compose restart suricata`
- [ ] 🟡 Reboot test: `PROMISC` flag present, offloads off, and the `suricata` / `ntopng` containers back up without manual steps
- [ ] 🟡 Mirror source and destination switch ports match what [Traffic Monitoring](../network/NET-08-Monitoring.md) records
- [ ] 🟡 Decide and document how ET Open rules are updated and how often
- [ ] DoH visibility: decide whether to block DoH at the router so DNS stays inspectable

## Related

- [SRV-26-TRBL](../troubleshooting/SRV-26-TRBL-Suricata-IDS.md) — troubleshooting for this doc
- [AI Alert Triage](./SRV-27-AI-Alert-Triage.md) — turns these alerts into a daily report
- [Traffic Monitoring](../network/NET-08-Monitoring.md) — switch mirror configuration and ntopng, which reads the same NIC
- [Docker Installation](./SRV-04-Docker-Installation.md)
- Guide: [Port Mirroring and Promiscuous Mode](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/operations/Guide-Port-Mirroring-and-Promiscuous-Mode.md)
- Guide: [IDS vs IPS and Blind Spots](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/operations/Guide-IDS-vs-IPS-and-Blind-Spots.md)
- Guide: [Reading Suricata Alerts](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/operations/Guide-Reading-Suricata-Alerts.md)
