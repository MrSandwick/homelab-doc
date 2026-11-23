---
tags: [homelab, project, kubernetes, minecraft, troubleshooting]
---

# Troubleshooting — Minecraft Server

> Companion to [Minecraft Server](../server/SRV-29-Minecraft-Server.md), which records only the working procedure. This file records the problems hit during that work.

## Contents

| Incident | Step |
|---|---|
| [`helm install` fails on a comma in the whitelist](#helm-install-fails-on-a-comma-in-the-whitelist) | [Step 3](../server/SRV-29-Minecraft-Server.md#step-3--server) |
| [Schema rejects `onlineMode: "TRUE"`](#schema-rejects-onlinemode-true) | [Step 3](../server/SRV-29-Minecraft-Server.md#step-3--server) |
| [Server stops on a whitelist name](#server-stops-on-a-whitelist-name) | [Step 4](../server/SRV-29-Minecraft-Server.md#step-4--whitelist-and-operator) |
| [`container not found` from `rcon-cli`](#container-not-found-from-rcon-cli) | [Step 4](../server/SRV-29-Minecraft-Server.md#step-4--whitelist-and-operator) |
| [AuthMe config missing right after the upgrade](#authme-config-missing-right-after-the-upgrade) | [Step 5](../server/SRV-29-Minecraft-Server.md#step-5--authme-and-sessions) |
| [Empty query right after adding the probe](#empty-query-right-after-adding-the-probe) | [Step 7](../server/SRV-29-Minecraft-Server.md#step-7--probe-and-alert) |

## `helm install` fails on a comma in the whitelist

**Symptom:**

```
Error: INSTALLATION FAILED: failed parsing --set-string data: key "<PLAYER>" has no value (cannot end with ,)
```

**Root cause:** `--set` and `--set-string` use commas to separate keys, so `whitelist="a,b,c"` was read as three settings.

**Fix:** escape the commas (`a\,b\,c`; in bash `${WL//,/\\,}`). The whitelist was later moved out of the chart altogether (see below).

## Schema rejects `onlineMode: "TRUE"`

**Symptom:**

```
values don't meet the specifications of the schema(s) ...
- at '/minecraftServer/onlineMode': got string, want boolean
```

**Root cause:** the chart validates its values against a JSON schema. `onlineMode` accepts `default` or a boolean, not the string `"TRUE"` that the image's own environment variable uses.

**Fix:** `onlineMode: true` (later `false`). A `helm install … --dry-run` now runs before every real install, so schema errors appear before anything is created.

## Server stops on a whitelist name

**Symptom:** the pod in `CrashLoopBackOff`; the log ends with:

```
[mc-image-helper] ERROR : Invalid parameter provided for 'manage-users' command: Could not resolve user from Playerdb: <PLAYER>
```

It continued after switching to offline mode.

**Root cause:** the image's start script turns the chart's whitelist and operator lists into `whitelist.json` and `ops.json` by looking each name up in Playerdb, a database of paid accounts. A player without one is not there, and the script stops. Offline mode does not skip the lookup.

**Fix:** the whitelist and operators were removed from the chart values and the release upgraded with `--reset-values`, so the earlier `--set` values were dropped too. The server then started, and the list was built in game with `rcon-cli whitelist add`, which computes the offline identifier from the name. A whitelist works in offline mode; only the image's pre-start lookup does not.

## `container not found` from `rcon-cli`

**Symptom:** every `kubectl exec … rcon-cli` returned `unable to upgrade connection: container not found ("minecraft")`.

**Root cause:** the container was between restarts. The upgrade that removed the whitelist had not been run yet, so the pod was still crashing on the old values (`helm history` showed revision 2, not 3).

**Fix:** run the upgrade, then wait for readiness before any `rcon-cli`: `kubectl -n minecraft wait --for=condition=Ready pod -l app.kubernetes.io/name=minecraft --timeout=300s`.

## AuthMe config missing right after the upgrade

**Symptom:** the plugin jar had been downloaded, but `rcon-cli` answered `connection refused` and `plugins/AuthMe/config.yml` did not exist.

**Root cause:** the server was still starting. Plugins write their config when they are enabled, after the world loads; RCON opens at the same point.

**Fix:** wait for readiness (the `kubectl wait` above), then read the config.

## Empty query right after adding the probe

**Symptom:** a Prometheus query for `probe_success{instance="mc.<DOMAIN>:25565"}` returned nothing for the first minutes.

**Root cause:** the operator first regenerates Prometheus's configuration from the new `Probe`, the config reloader applies it, and only then is the first probe scraped.

**Fix:** none needed; the series appeared within three minutes with value `1`. A loop that queries every ten seconds for up to three minutes avoids re-running the check by hand.
