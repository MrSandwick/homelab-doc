---
tags: [homelab-project, homelab, note, project, networking, monitoring, troubleshooting]
---

# Troubleshooting — Traffic Monitoring

> Companion to [Traffic Monitoring](../network/NET-08-Monitoring.md), which records only the working configuration. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Relates to |
|---|---|
| [`ntop/ntopng:stable` tag does not exist](#ntopntopngstable-tag-does-not-exist) | [ntopng](../network/NET-08-Monitoring.md#ntopng-docker-with-redis) |
| [NIC offloading corrupts monitored packet sizes](#nic-offloading-corrupts-monitored-packet-sizes) | [Port mirroring](../network/NET-08-Monitoring.md#network-wide-visibility-switch-port-mirroring) |
| ["Ghost Networks" alerts](#ghost-networks-alerts) | [ntopng](../network/NET-08-Monitoring.md#ntopng-docker-with-redis) |
| [Monitoring interface showed `NO-CARRIER`](#monitoring-interface-showed-no-carrier) | [Port mirroring](../network/NET-08-Monitoring.md#network-wide-visibility-switch-port-mirroring) |
| [Forgotten ntopng admin password](#forgotten-ntopng-admin-password) | [ntopng](../network/NET-08-Monitoring.md#ntopng-docker-with-redis) |

## `ntop/ntopng:stable` tag does not exist

**Symptom:** the first attempt used `image: ntop/ntopng:stable`, which failed with `failed to resolve reference ... not found`.

**Root cause:** the official image is currently only published under the `:latest` tag on Docker Hub.

**Fix:** switched to `ntop/ntopng:latest`.

## NIC offloading corrupts monitored packet sizes

**Symptom:** ntopng's startup log flagged an accuracy problem on the mirrored interface:

```
Packets exceeding the expected max size have been received [enp2s0][len: 1646][max len: 1518]
WARNING: If TSO/GRO is enabled, please disable it for best accuracy
```

**Root cause:** hardware offloading features (TSO/GSO on transmit, GRO on receive) merge multiple real packets into one larger "virtual" packet before software sees them — a performance optimization that is harmful on a monitoring interface, since ntopng needs genuine per-packet boundaries and timing. A merged "packet" of 1646 bytes is larger than Ethernet's real 1518-byte maximum.

**Fix:** offloading disabled on `enp2s0` and made persistent with a systemd unit — see [Port mirroring](../network/NET-08-Monitoring.md#network-wide-visibility-switch-port-mirroring).

## "Ghost Networks" alerts

**Symptom:** two persistent `Ghost Networks` alerts, for the management subnet and the Admin subnet, plus a banner reading `No local hosts detected, although there are active hosts. Please make sure local networks have been properly configured (-m parameter).`

**Root cause:** the monitored interface (`enp2s0`) has no IP address of its own (by design — it is a passive mirror), so ntopng cannot infer which subnets are "local" from the interface's own address. Every subnet it observes traffic from gets flagged as unexpected.

**Fix:** the known local subnets were declared with the `-m` startup parameter in `docker-compose.yml` (now part of the compose file in the main doc), then:

```
docker compose up -d --force-recreate ntopng
```

## Monitoring interface showed `NO-CARRIER`

**Symptom:** the ntopng dashboard showed no traffic at all (0 bps, empty `Hosts` list), and `ip link show enp2s0` reported:

```
<NO-CARRIER,BROADCAST,MULTICAST,PROMISC,UP> ... state DOWN
```

**Root cause:** a purely physical-layer issue, unrelated to any ntopng or switch configuration — `NO-CARRIER` means the interface detects no electrical signal from the switch. The monitoring cable had become disconnected (or was never fully seated) at one end, likely disturbed while re-cabling the switch to accommodate the second and third cluster nodes joining around the same time (see [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md) for the port reassignments in that period).

**Fix:** reseated the cable between the server's `enp2s0` NIC and the switch's mirroring port. `PROMISC` and `UP` being already set did not matter — without `LOWER_UP` (physical link detected), no traffic can arrive regardless of any mirroring or interface configuration.

**Takeaway:** `NO-CARRIER` is the first thing to rule out whenever a monitoring/mirrored interface "stops working" after cabling changes nearby, before suspecting the Port Mirror configuration or ntopng.

## Forgotten ntopng admin password

**What happened:** while troubleshooting the above, the ntopng admin password set during initial setup had been forgotten. ntopng and Grafana (the cluster-monitoring dashboard on the Kubernetes side) were briefly confused, since both are web dashboards with an `admin` account.

**Fix:** reset ntopng's local user database:

```
cd ~/ntopng
docker compose down
rm -rf ./data/*
docker compose up -d
```

This clears ntopng's accumulated traffic history along with its user database — acceptable here since the history was not load-bearing, but a real cost of this reset method. Logged back in with the default (`admin`/`admin`) and changed the password immediately afterward.

## Related

- [Traffic Monitoring](../network/NET-08-Monitoring.md) — the working configuration
- [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md) — the port reassignments around the cable fault
- Guide: [Port Mirroring and Promiscuous Mode](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/Guide-Port-Mirroring-and-Promiscuous-Mode.md)
