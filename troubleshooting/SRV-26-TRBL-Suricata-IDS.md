---
tags: [homelab, project, security, ids, suricata, troubleshooting]
---

# Troubleshooting — Suricata IDS

> Companion to [Suricata IDS](../server/SRV-26-Suricata-IDS.md), which records only the working configuration. This file records the problems hit during that work, in the order they occurred.

## Contents

1. [BPF filter rejecting every packet](#bpf-filter-rejecting-every-packet)
2. [Decoder alerts from link-layer management frames](#decoder-alerts-from-link-layer-management-frames)
3. [`Permission denied` reading the rules files](#permission-denied-reading-the-rules-files)
4. [logrotate `Permission denied` in a user home directory](#logrotate-permission-denied-in-a-user-home-directory)
5. [`testmynids.org` generates no alert](#testmynidsorg-generates-no-alert)

## BPF filter rejecting every packet

**Symptom:** the container looked like it had started, but no events ever reached `eve.json`. `docker inspect` showed it had restarted 13 times.

**Root cause:** the first version of the capture filter lost its `not` and its parentheses. The remaining expression matched nothing, Suricata exited with `expression rejects all packets`, and `restart: unless-stopped` put it straight into a restart loop. `docker compose up -d` had reported success, because it only reports that the container was *created* — a crash a second later does not change its exit code.

**Fix:** validate the expression before using it, and check the container state rather than the exit code of `up -d`:

```
sudo tcpdump -d -i enp2s0 'not (ether proto 0x8899 or ether proto 0x890d or ether proto 0x88cc)'
docker inspect -f '{{.State.Status}} restarts={{.RestartCount}}' suricata
```

`tcpdump -d` prints BPF bytecode only when the expression parses, so a syntax error surfaces before Suricata ever sees it. The healthy state is `running restarts=0`.

**Takeaway:** for any container with `restart: unless-stopped`, a crash loop is invisible unless you look — the compose command, and `docker ps` a moment later, both look normal at first glance.

## Decoder alerts from link-layer management frames

**Symptom:** a constant stream of "unknown Ethertype" decoder alerts, drowning real signatures.

**Root cause:** the mirror port carries everything the router-uplink port sees, including LLDP (`0x88cc`) and two other link-layer management protocols (`0x8899`, `0x890d`) spoken by the network gear itself. Suricata decodes each frame and complains about Ethertypes it has no decoder for.

**Fix:** drop them at capture time with the BPF filter, rather than suppressing the alerts afterwards — the frames carry no IP traffic, so nothing of interest is lost and the packets are never decoded in the first place.

## `Permission denied` reading the rules files

**Symptom:** `less lib/rules/suricata.rules` as the normal login user returns `Permission denied`, which looks like a broken rule download.

**Root cause:** the rules are written by the container's user, not by the host login user. The files are present and correct; only the read permission is missing.

**Fix:** read them with `sudo`. Nothing needs changing — this is expected for a bind-mounted volume written from inside a container.

## logrotate `Permission denied` in a user home directory

**Symptom:** logrotate failed to rotate `eve.json`, reporting `Permission denied`, even though it runs as root.

**Root cause:** two separate things. The log files belong to a host account named `systemd-network` (the uid the container writes as), and logrotate refuses to rotate files in a directory it considers unsafe to act in as root. Even rotating as the file owner failed, because that account could not traverse the home directory on the way to `~/suricata/log`.

**Fix:** tell logrotate to act as the owning account, and grant that account execute-only access to the home directory so it can traverse it:

```
su systemd-network systemd-network     # inside /etc/logrotate.d/suricata-docker
```

```
sudo setfacl -m u:systemd-network:x /home/<USERNAME>
sudo logrotate -f /etc/logrotate.d/suricata-docker
```

Execute-only (`x`, not `rx`) lets the account walk through the directory without being able to list its contents.

## `testmynids.org` generates no alert

**Symptom:** the standard IDS smoke test produced nothing at all — no alert, and no sign the request had happened.

**Root cause:** a typo in the hostname. The name used does not resolve, so no HTTP request was ever made and there was nothing for the sensor to see. An IDS that is working correctly and an IDS that is receiving no traffic look identical from the alert log.

**Fix:** use `curl http://testmyids.com` and confirm the expected signature arrives:

```
grep -c 'id check returned root' ~/suricata/log/eve.json
```

## Related

- [Suricata IDS](../server/SRV-26-Suricata-IDS.md) — the working configuration
- [NET-08-TRBL](./NET-08-TRBL-Monitoring.md) — problems with the same mirror NIC on the ntopng side
