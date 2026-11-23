---
tags: [homelab, project, kubernetes, minecraft, paper, authme, metallb, networkpolicy, blackbox-exporter, alerting]
---

# Minecraft Server

> Status: 🟢 **Working.** A Minecraft Java server (Paper) runs in the cluster and is reachable from the internet at `mc.<DOMAIN>`. Players without a paid account can join; each registers a password once with AuthMe, and a 30-day session per address spares them `/login` afterwards. The namespace accepts only the game port, and a probe of the public address alerts in Slack after five minutes down. 🔴 **No backup of the world yet** — see [Limits](#limits).

Picks up after [Cilium CNI Migration and NetworkPolicy](./SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md), which was done first so that the new namespace could have enforced rules before going public. Commands run on the control-plane node (`<HOSTNAME>`) unless noted. Problems hit along the way: [SRV-29-TRBL](../troubleshooting/SRV-29-TRBL-Minecraft-Server.md).

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Capacity and address](#step-1--capacity-and-address)
5. [Step 2 — Namespace and policy](#step-2--namespace-and-policy)
6. [Step 3 — Server](#step-3--server)
7. [Step 4 — Whitelist and operator](#step-4--whitelist-and-operator)
8. [Step 5 — AuthMe and sessions](#step-5--authme-and-sessions)
9. [Step 6 — Internet access](#step-6--internet-access)
10. [Step 7 — Probe and alert](#step-7--probe-and-alert)
11. [Operations](#operations)
12. [Verification](#verification)
13. [Limits](#limits)
14. [Files](#files)
15. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Helm release `minecraft` (chart `itzg/minecraft` 5.3.1, image `itzg/minecraft-server`) | Namespace `minecraft`, on the worker | Paper server, Java edition; the image downloads Paper and plugins at start |
| PVC `minecraft-datadir`, 10 Gi, `local-path` | Worker | World, `server.properties`, `whitelist.json`, `ops.json`, plugin data |
| Service `minecraft`, type `LoadBalancer` | MetalLB address `<MINECRAFT_IP>`, TCP 25565 | Game port, its own address separate from both Gateways |
| Service `minecraft-rcon`, `ClusterIP` | TCP 25575 | Remote console; used through `kubectl exec … rcon-cli` |
| AuthMe Reloaded 6.0.1 (Paper build) | Downloaded from Modrinth at start | Password per player name; sessions |
| NetworkPolicies `default-deny-ingress`, `allow-game-port` | `minecraft` | Only TCP 25565 is accepted |
| Blackbox module `public_tcp`, `Probe minecraft-public`, `PrometheusRule minecraft` | `monitoring` | TCP check of `mc.<DOMAIN>:25565` every minute; `MinecraftServerDown` |
| Port forwards (ISP gateway, ER605) and `CNAME mc` | Routers, Cloudflare | Public path, the same chain as the site ([NET-09](../network/NET-09-Public-Port-Forwarding.md)) |

```
Player → mc.<DOMAIN> (CNAME → <DOMAIN>) → <PUBLIC_IP>:25565 → ISP gateway → ER605 → <MINECRAFT_IP>:25565 → pod
```

## Why

A server for a small group of friends was on the list of side projects from the start ([SRV-00](./SRV-00-Project-Overview.md)). It is also the first stateful workload in the cluster that strangers on the internet talk to directly, without a Gateway in front: a good test of the network policy, the public path and the alerting built earlier.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Image and chart | `itzg/minecraft-server` with the `itzg/minecraft` chart | The standard way to run Minecraft in containers; server type, version and plugins are environment variables |
| Server type | Paper | Better performance than vanilla and the usual base for plugins |
| Edition | Java | What the players use |
| Accounts | Offline mode (`onlineMode: false`) | Not every player has a paid account. Without the Mojang check, a name is not proof of identity |
| Identity | AuthMe: register once, sessions for 30 days per address | Without it anyone who knows a player's name could join as that player. A VPN and licensed accounts were the alternatives; both were rejected as too much friction for the players |
| Operator rights | Given to one name, used rarely | In offline mode an operator name is only as safe as its AuthMe password; admin work can also be done with `rcon-cli` from the control plane |
| Whitelist | On, managed in game through `rcon-cli`, not through the chart | The image checks chart whitelist names against a database of paid accounts and stops on any other name ([SRV-29-TRBL](../troubleshooting/SRV-29-TRBL-Minecraft-Server.md#server-stops-on-a-whitelist-name)) |
| Placement | The worker, without a `nodeSelector` | The control plane and the laptop are tainted, so the worker is the only possible node; real node names stay out of the public repository |
| Memory | Request and limit 4 Gi, heap 3 GB, Aikar's JVM flags | Ten players with view distance 8; the rest of the limit is for the JVM outside the heap. Fits on the worker (about 5 Gi requested before) |
| Storage | 10 Gi on `local-path` | The world lives on the worker's disk, like every other volume |
| Address | Its own MetalLB address | Neither Gateway is involved; a separate address keeps the game port apart from the site |
| Client address | `externalTrafficPolicy: Local` | The server and AuthMe see the player's real address instead of a node address |
| RCON | Enabled with a generated password, not exposed | Needed for the whitelist and operators; reachable only from inside the pod |
| Ingress policy | Default deny, TCP 25565 only | RCON and anything else in the namespace stay unreachable even from other pods |
| Settings in Git | Values, policy and monitoring only | The MetalLB address is passed at install; whitelist, operators and the AuthMe config live on the volume |
| Monitoring | TCP probe of the public name | One check covers DNS, both forwards, MetalLB and the server, the same approach as the site in [SRV-25](./SRV-25-Public-Site-Operations.md) |
| Backups | Not yet | Deferred by decision; see [Limits](#limits) |

## Step 1 — Capacity and address

```
kubectl describe node <WORKER_HOSTNAME> | sed -n '/Allocated resources/,/Events/p'
kubectl get svc -A | grep LoadBalancer
kubectl -n metallb-system get ipaddresspool -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.addresses}{"\n"}{end}'
```

The worker had about a third of its memory requested, so 4 Gi more fit. The MetalLB pool had two addresses in use (the LAN Gateway and the public Gateway); the next free one became `<MINECRAFT_IP>`.

The chart's options were read from the chart itself rather than from memory:

```
helm repo add itzg https://itzg.github.io/minecraft-server-charts/
helm show values itzg/minecraft > /tmp/mc-values.yaml
grep -v -E '^\s*#' /tmp/mc-values.yaml | grep -v '^\s*$'
```

## Step 2 — Namespace and policy

`platform/minecraft/namespace-policy.yaml` creates the namespace with its rules before anything runs in it:

```yaml
# Namespace with a default-deny ingress policy; only the game port is open.
# RCON (25575) is not opened: nothing outside the pod can reach it.
apiVersion: v1
kind: Namespace
metadata:
  name: minecraft
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: minecraft
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-game-port
  namespace: minecraft
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - port: 25565
          protocol: TCP
```

```
kubectl apply -f platform/minecraft/namespace-policy.yaml
```

Cilium applies `ipBlock` (CIDR) rules to traffic from outside the cluster, which is where players and the probe come from. Whether pods in other namespaces can reach the game port was not tested.

## Step 3 — Server

`platform/minecraft/values.yaml` (final form):

```yaml
# itzg/minecraft chart 5.3.1: Paper server (Java edition) for friends over the internet.
# The pod can only land on the worker: the control plane and the GPU laptop are tainted.
# Not stored here (passed with --set at install): LoadBalancer address.
image:
  repository: itzg/minecraft-server
  tag: latest
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: 4Gi
    cpu: 1000m
  limits:
    memory: 4Gi

terminationGracePeriodSeconds: 60   # time to save the world on shutdown

startupProbe:
  enabled: true
  failureThreshold: 60              # first start downloads Paper and generates the world
  periodSeconds: 10

minecraftServer:
  eula: "TRUE"                      # accepts https://aka.ms/MinecraftEULA
  type: "PAPER"
  version: "LATEST"                 # pin after the first start
  maxPlayers: 10
  difficulty: normal
  gameMode: survival
  motd: "Homelab Minecraft"
  viewDistance: 8
  onlineMode: false                 # non-licensed clients allowed; AuthMe required
  enforceSecureProfile: false
  memory: 3G                        # JVM heap; the rest of the 4Gi limit is off-heap memory
  serviceType: LoadBalancer
  extraServiceSpec:
    externalTrafficPolicy: Local    # keep the real client address
  modrinth:
    projects:
      - authmereloaded              # register once, sessions keep players logged in
  rcon:
    enabled: true                   # cluster-internal only, never exposed
    withGeneratedPassword: true

extraEnv:
  USE_AIKAR_FLAGS: "true"           # recommended JVM flags for Paper

persistence:
  dataDir:
    enabled: true
    Size: 10Gi
```

`eula: "TRUE"` accepts the Minecraft EULA; that is the owner's decision, not a default. The chart's schema wants booleans for `onlineMode` and `enforceSecureProfile`, and a dry run catches type errors before anything is created:

```
helm install minecraft itzg/minecraft --version 5.3.1 -n minecraft -f platform/minecraft/values.yaml \
  --set-string minecraftServer.loadBalancerIP="<MINECRAFT_IP>" --dry-run > /dev/null && echo "DRY RUN OK"

helm install minecraft itzg/minecraft --version 5.3.1 -n minecraft -f platform/minecraft/values.yaml \
  --set-string minecraftServer.loadBalancerIP="<MINECRAFT_IP>"
```

Later changes use `helm upgrade … --reset-values` with the same arguments, so nothing passed with `--set` on an earlier run survives by accident.

```
kubectl -n minecraft get pods,svc,pvc
kubectl -n minecraft logs deploy/minecraft --tail=20
```

The pod becomes `1/1` once the server prints `Done (…)! For help, type "help"`; the first start downloads Paper and creates the world. The Service shows `EXTERNAL-IP <MINECRAFT_IP>`, the volume `Bound`. Rather than polling by hand, one command waits for readiness:

```
kubectl -n minecraft wait --for=condition=Ready pod -l app.kubernetes.io/name=minecraft --timeout=300s
```

## Step 4 — Whitelist and operator

The whitelist is managed by the server itself, which computes the offline identifier of each name:

```
kubectl -n minecraft exec deploy/minecraft -- rcon-cli whitelist on
kubectl -n minecraft exec deploy/minecraft -- rcon-cli whitelist add <PLAYER>
kubectl -n minecraft exec deploy/minecraft -- rcon-cli whitelist list
kubectl -n minecraft exec deploy/minecraft -- rcon-cli op <PLAYER>
```

In offline mode a whitelist limits **names**, not people: it keeps scanners and strangers out, but anyone who knows a listed name could use it. That is what AuthMe closes.

## Step 5 — AuthMe and sessions

AuthMe Reloaded is the most used authentication plugin for Paper. Version 6.0.1 or newer is required: it fixed a session takeover on Paper servers in offline mode. The chart downloads it from Modrinth (`minecraftServer.modrinth.projects`), choosing the latest release for Paper and the server's version; the log shows `Downloaded /data/plugins/AuthMe-6.0.1-Paper.jar`.

Plain AuthMe asks for `/login` at every join. Sessions skip it while the player comes back from the same address within the timeout. The config file is created at the first start, on the volume:

```
kubectl -n minecraft exec deploy/minecraft -- grep -n -A12 'sessions:' /data/plugins/AuthMe/config.yml
```

Under `settings.sessions`: `enabled: false` → `true`, `timeout: 10` → `43200` (minutes, 30 days). Edited in place and reloaded without restarting the server:

```
kubectl -n minecraft exec deploy/minecraft -- sed -i \
  -e '<line>s/enabled: false/enabled: true/' \
  -e '<line>s/timeout: 10/timeout: 43200/' \
  /data/plugins/AuthMe/config.yml
kubectl -n minecraft exec deploy/minecraft -- rcon-cli authme reload
```

Line numbers come from the `grep` above. For the players: `/register <password> <password>` once; afterwards they join without typing anything, until their address changes or 30 days pass. AuthMe warns at start that no MaxMind key is configured; GeoIP checks are not used.

## Step 6 — Internet access

The same chain as the site ([NET-09](../network/NET-09-Public-Port-Forwarding.md)), one more rule on each router:

| Where | Rule |
|---|---|
| ISP gateway → Port Forwarding | `minecraft`, TCP 25565 → `<ER605_WAN_IP>`:25565, schedule *Always*, then **Apply Changes** |
| ER605 → Settings → Transmission → NAT → Virtual Servers | `minecraft`, WAN, 25565 → `<MINECRAFT_IP>`:25565, TCP |
| Cloudflare DNS | `CNAME mc → <DOMAIN>`, **DNS only** |

The `CNAME` follows the apex record, which the DNS updater from [SRV-25](./SRV-25-Public-Site-Operations.md) keeps on the current public address, so no second updater is needed. Cloudflare's proxy does not carry Minecraft traffic, so the record must stay grey. Players connect to `mc.<DOMAIN>` without a port; 25565 is the default.

Checked from a phone on mobile data with a public server-status page: online, version 26.3.

## Step 7 — Probe and alert

A TCP module for the existing blackbox exporter, added to `platform/blackbox-exporter/values.yaml` under `config.modules`:

```yaml
    public_tcp:
      prober: tcp
      timeout: 10s
      tcp:
        preferred_ip_protocol: ip4
        ip_protocol_fallback: false
```

```
helm upgrade blackbox prometheus-community/prometheus-blackbox-exporter -n monitoring \
  --version <installed chart version> -f platform/blackbox-exporter/values.yaml
```

`platform/minecraft/monitoring.yaml` holds the `Probe` and the rule, both labelled `release: prometheus` so this Prometheus picks them up:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: minecraft-public
  namespace: monitoring
  labels:
    release: prometheus
spec:
  interval: 60s
  module: public_tcp
  prober:
    url: blackbox-prometheus-blackbox-exporter.monitoring.svc:9115
  targets:
    staticConfig:
      static:
        - mc.<DOMAIN>:25565
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: minecraft
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: minecraft
      rules:
        - alert: MinecraftServerDown
          expr: probe_success{instance="mc.<DOMAIN>:25565"} == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: Minecraft server is unreachable from the internet
            description: TCP connection to mc.<DOMAIN>:25565 has failed for 5 minutes (DNS, port forwards, MetalLB or the server pod).
```

The probe goes out to the public address and back in through both routers, the same hairpin the site's probe relies on. It arrives from outside the cluster, so the namespace's `ipBlock` rule lets it in. Result after the operator reloaded Prometheus (one to three minutes):

```
probe/monitoring/minecraft-public mc.<DOMAIN>:25565 1
probe_success 1
```

The alert is `warning`, so Alertmanager repeats it every 48 hours ([SRV-22](./SRV-22-Alerting-Reliability.md#step-5--fewer-repeats)).

## Operations

| Task | Command |
|---|---|
| Add a player | `kubectl -n minecraft exec deploy/minecraft -- rcon-cli whitelist add <PLAYER>` |
| Reset a player's password | `kubectl -n minecraft exec deploy/minecraft -- rcon-cli authme unregister <PLAYER>`; they register again at the next join |
| Any server command | `kubectl -n minecraft exec deploy/minecraft -- rcon-cli <command>` |
| Logs | `kubectl -n minecraft logs deploy/minecraft --tail=50` |
| Change settings | Edit `values.yaml`, then `helm upgrade … --reset-values` with the MetalLB address as in Step 3 |

## Verification

```
kubectl -n minecraft get pods,svc,pvc,networkpolicy
kubectl -n minecraft exec deploy/minecraft -- rcon-cli plugins
kubectl -n minecraft exec deploy/minecraft -- grep -E 'online-mode|white-list' /data/server.properties
kubectl -n monitoring get probe minecraft-public
```

Pod `1/1`, Service on `<MINECRAFT_IP>`, volume `Bound`, two policies; `AuthMe` in the plugin list; `online-mode=false`, `white-list=true`. In game: a listed name registers once and rejoins without `/login`; a name not on the list is refused.

## Limits

- **No backup of the world.** The world exists only on the worker's disk. A failed disk or a deleted volume loses it. The `itzg/mc-backup` sidecar is already part of the chart (`mcbackup`), and backing up to cloud storage is the next step.
- **Offline mode trusts AuthMe.** A player's identity is their AuthMe password. A session is reused from the same address for 30 days, so someone behind the same address (the same home or carrier NAT) could join under that name during the session.
- **Operator rights follow a name.** Their safety equals that name's AuthMe password.
- **The home address is public.** `mc.<DOMAIN>` resolves to the house's address and the port is open to everyone; whitelist and AuthMe limit who can play, not who can connect.
- **`latest` image and server version.** A restart can bring a newer Minecraft or Paper; players then need the matching client. Pinning both is an open item.
- **Settings outside Git.** Whitelist, operators and the AuthMe config are on the volume. They are lost with it and are not reviewed in pull requests.
- **One worker.** The volume is local, so the server cannot move to another node.
- **The probe shares the house.** If the internet connection is down, nothing reports; the same limit as everywhere else ([SRV-22](./SRV-22-Alerting-Reliability.md#limits)).

## Files

| File | Location | Contents |
|---|---|---|
| `values.yaml` | `platform/minecraft/` in `homelab-gitops` | Chart values |
| `namespace-policy.yaml` | `platform/minecraft/` | Namespace and the two policies |
| `monitoring.yaml` | `platform/minecraft/` | `Probe` and `PrometheusRule` |
| `values.yaml` | `platform/blackbox-exporter/` | Now also the `public_tcp` module |
| `whitelist.json`, `ops.json`, `plugins/AuthMe/` | Volume `minecraft-datadir`, not in Git | Players, operators, AuthMe config and accounts |

## Related

- [Public Port Forwarding](../network/NET-09-Public-Port-Forwarding.md) — the router chain this reuses
- [Public Site Operations](./SRV-25-Public-Site-Operations.md) — DNS updater and the blackbox exporter
- [Cilium CNI Migration and NetworkPolicy](./SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md) — why the namespace policy is enforced
- [itzg/docker-minecraft-server documentation](https://docker-minecraft-server.readthedocs.io/) — image settings
- [AuthMe Reloaded on Modrinth](https://modrinth.com/plugin/authmereloaded)
