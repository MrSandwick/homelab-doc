---
tags: [homelab, project, kubernetes, ddns, cloudflare, blackbox-exporter, prometheus, alerting, rate-limiting, backup, mongodb]
---

# Public Site Operations

> Status: 🟢 **Working.** A CronJob keeps the DNS record on the connection's current public address; a blackbox exporter checks the public site every minute and three alerts reach Slack; a per-visitor rate limit sits at the Gateway; the nightly backup includes the database. The rate limit was tested with a flood (626 requests passed, 174 were answered `429`), the updater and the probe with live runs. 🟡 Limits: the monitor lives in the same house as the site, the backup copies are in the same house too (decided), and a restore of the Mongo dump has not been tested yet — see [Limits](#limits).

Picks up after [Portfolio Hosting](./SRV-24-Portfolio-Hosting.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the admin user `<USER>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Keep the DNS record on the current address](#step-1--keep-the-dns-record-on-the-current-address)
5. [Step 2 — Probe and alerts](#step-2--probe-and-alerts)
6. [Step 3 — Rate limit at the Gateway](#step-3--rate-limit-at-the-gateway)
7. [Step 4 — Back up the database](#step-4--back-up-the-database)
8. [Verification](#verification)
9. [Limits](#limits)
10. [Files](#files)
11. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| CronJob `ddns` (every 5 minutes) | `portfolio`, image `curlimages/curl` | Reads the public IPv4 address, compares it with the `A` record of `<DOMAIN>` and updates the record only when they differ |
| Secret `cloudflare-ddns` | `portfolio`, not in Git | `CF_API_TOKEN` (permission *Zone DNS: Edit*, this one zone only) and `CF_ZONE_ID` |
| Helm release `blackbox` (chart `prometheus-community/prometheus-blackbox-exporter`) | `monitoring` | Opens `https://<DOMAIN>/` and `/api/health` the way a visitor would; module `public_https` |
| `Probe` `<DOMAIN_SLUG>-public` | `monitoring` | Tells Prometheus to ask the exporter every 60 seconds |
| `PrometheusRule` `portfolio` | `monitoring` | `PortfolioSiteDown`, `PortfolioCertificateExpiring`, `PortfolioDdnsNotRunning` |
| `BackendTrafficPolicy` `portfolio-limits` | `portfolio` | Envoy local rate limit: 600 requests per minute per visitor address on the site route |
| `backup_mongo` step | `backup-homelab.sh` on the control plane | `mongodump` of the `portfolio` database into the nightly backup folder |

## Why

The connection's public address is dynamic. After a change the name would point at nothing until someone edited the record. And once the site is public, three more things are needed that a LAN-only lab did without: someone has to notice when it is unreachable, a flood of requests must not reach nginx and the API unchecked, and the data has to be in the backups.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Address updater | A CronJob in the cluster with the DNS provider's API | The ISP gateway has a Dynamic DNS page, but it was not used: a CronJob needs no router support, works with any DNS host that has an API, and is visible in the same GitOps and alerting setup as everything else |
| Where the address comes from | Cloudflare's trace page (`https://cloudflare.com/cdn-cgi/trace`) | A plain HTTPS request; the ISP blocks `1.1.1.1` and `8.8.8.8` by address, not this name. One more outbound lookup at a third party, with no credentials |
| What the updater changes | Only the `content` of the `A` record (PATCH), never the proxy setting; `www` is a CNAME to the apex, so it follows | A single record to keep in sync; a mistake cannot flip the proxy |
| Token scope | One zone, DNS edit only | A leaked token can change DNS for one domain and nothing else |
| Interval | 5 minutes; Cloudflare TTL Auto (about 5 minutes) | Worst case about 10 minutes of downtime after an address change |
| What is probed | The public name from inside the cluster | The router passes traffic from the LAN to its own public address (no hairpin problem, checked with `curl`), so the probe covers DNS, the ISP gateway, the ER605, the Gateway, certificate, nginx and API in one |
| Alert thresholds | Site down for 5 minutes (critical); certificate under 14 days for 1 hour (warning); no successful updater run for 30 minutes (warning) | Minutes of flapping do not page; a stuck updater is noticed long before the address changes |
| Rate limit | Local (per Envoy pod), 600 requests per minute per visitor address | Far above a page load with the 3D models (well under 100 requests), far below a flood; the API keeps its own stricter limits (120 per minute, 8 for the contact form) |
| Mongo backup | An **optional** step in the existing script | A database problem gives `PARTIAL` and a Slack message, but does not throw away the good etcd and certificate backups of that night |
| Off-site backups | None, by decision | The rule of the project is no third-party storage; the copies are on the control plane and the worker |

## Step 1 — Keep the DNS record on the current address

A Cloudflare API token from the *Edit zone DNS* template, restricted to the one zone, plus the zone ID from the zone's overview page, are stored as a Secret (read with a hidden prompt, so neither lands in the shell history or a file):

```
read -s -p "Cloudflare API token: " T; echo; read -p "Zone ID: " Z
kubectl -n portfolio create secret generic cloudflare-ddns --from-literal=CF_API_TOKEN="$T" --from-literal=CF_ZONE_ID="$Z"; unset T Z
```

`apps/portfolio/ddns.yaml` in `homelab-gitops`: a CronJob (`concurrencyPolicy: Forbid`, `activeDeadlineSeconds: 60`, Pod Security `restricted`, read-only root filesystem, user 100) whose script does:

```sh
IP=$(curl -fsS --max-time 10 https://cloudflare.com/cdn-cgi/trace | sed -n 's/^ip=//p')
case "$IP" in *.*.*.*) ;; *) echo "no usable IPv4: $IP"; exit 1 ;; esac
API="https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records"
AUTH="Authorization: Bearer $CF_API_TOKEN"
REC=$(curl -fsS --max-time 15 -H "$AUTH" "$API?type=A&name=<DOMAIN>")
CUR=$(echo "$REC" | grep -o '"content":"[^"]*"' | head -1 | cut -d'"' -f4)
ID=$(echo "$REC" | grep -o '"id":"[^"]*"' | head -1 | cut -d'"' -f4)
[ -n "$ID" ] || { echo "A record not found"; exit 1; }
if [ "$CUR" = "$IP" ]; then echo "unchanged: $IP"; exit 0; fi
curl -fsS --max-time 15 -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
  --data "{\"content\":\"$IP\"}" "$API/$ID" > /dev/null
echo "updated: $CUR -> $IP"
```

Test once by hand after ArgoCD has synced it:

```
kubectl -n portfolio create job --from=cronjob/ddns ddns-test && sleep 15 && kubectl -n portfolio logs job/ddns-test
kubectl -n portfolio delete job ddns-test
```

`unchanged: <PUBLIC_IP>`.

## Step 2 — Probe and alerts

```
helm repo update prometheus-community
helm upgrade --install blackbox prometheus-community/prometheus-blackbox-exporter -n monitoring \
  --version <chart version> -f platform/blackbox-exporter/values.yaml
kubectl -n monitoring get pods,svc | grep blackbox
```

`platform/blackbox-exporter/values.yaml` defines the module `public_https` (HTTP GET, status 200 only, no redirects followed, TLS required, IPv4). The Service is `blackbox-prometheus-blackbox-exporter` on port 9115. A hand test through the Service shows the chain end to end:

```
curl -s "http://<EXPORTER_SERVICE_IP>:9115/probe?module=public_https&target=https://<DOMAIN>/api/health" \
  | grep -E "^probe_(success|http_status_code|ssl_earliest_cert_expiry)"
```

`probe_success 1`, `probe_http_status_code 200` and the certificate expiry as a timestamp.

`apps/portfolio/monitoring.yaml` holds the `Probe` (targets `https://<DOMAIN>/` and `https://<DOMAIN>/api/health`, interval 60 s) and the `PrometheusRule`. Both objects carry the label `release: prometheus`, which is what this Prometheus selects (the same label as `my-site-alerts`):

| Alert | Expression | For | Severity |
|---|---|---|---|
| `PortfolioSiteDown` | `probe_success{instance=~"https://<DOMAIN>.*"} == 0` | 5 m | critical |
| `PortfolioCertificateExpiring` | `probe_ssl_earliest_cert_expiry{instance="https://<DOMAIN>/"} - time() < 14 * 86400` | 1 h | warning |
| `PortfolioDdnsNotRunning` | `time() - kube_cronjob_status_last_successful_time{namespace="portfolio",cronjob="ddns"} > 1800` | 5 m | warning |

Check that Prometheus loaded them (the container has no `wget`, so the query goes through the Service address; the operator needs one to three minutes to reload after the objects appear):

```
PIP=$(kubectl -n monitoring get svc prometheus-kube-prometheus-prometheus -o jsonpath='{.spec.clusterIP}')
curl -s "http://$PIP:9090/api/v1/rules?type=alert" | grep -o '"name":"Portfolio[A-Za-z]*"' | sort -u
curl -s "http://$PIP:9090/api/v1/query" --data-urlencode 'query=probe_success{instance=~"https://<DOMAIN>.*"}'
```

All three rule names are listed and the probe reports `"1"`.

## Step 3 — Rate limit at the Gateway

`apps/portfolio/limits.yaml`:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: portfolio-limits
  namespace: portfolio
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: portfolio
  rateLimit:
    type: Local
    local:
      rules:
        - clientSelectors:
            - sourceCIDR:
                type: Distinct
                value: 0.0.0.0/0
          limit:
            requests: 600
            unit: Minute
```

`Distinct` gives every source address its own bucket. The visitor's real address is visible because the public Gateway keeps it (`externalTrafficPolicy: Local`, [SRV-23](./SRV-23-Public-Gateway-and-HTTPS.md)). Before applying, the CRD and the field were checked with `kubectl explain backendtrafficpolicy.spec.rateLimit.local.rules.clientSelectors`.

Flood test (from the control plane; its own address is throttled for about a minute):

```
seq 800 | xargs -P 25 -I{} curl -s -o /dev/null -w "%{http_code}\n" https://<DOMAIN>/healthz | sort | uniq -c
```

```
    626 200
    174 429
```

## Step 4 — Back up the database

The existing script ([SRV-22](./SRV-22-Alerting-Reliability.md#step-4--resilient-backup-script)) gets one more function and one more `step` line:

```bash
backup_mongo() {
  kubectl -n portfolio exec mongo-0 -- sh -c 'mongodump --db portfolio --archive --gzip -u "$APP_USER" -p "$APP_PASS" --authenticationDatabase portfolio' > "$OUT/mongo-portfolio.archive.gz" && [ -s "$OUT/mongo-portfolio.archive.gz" ] && return 0
  rm -f "$OUT/mongo-portfolio.archive.gz"
  return 1
}

step optional "MongoDB (portfolio)"  backup_mongo
```

The credentials are the pod's own environment (`APP_USER`, `APP_PASS` from the Secret), so nothing is written into the script. The dump is streamed out of the pod; a failed or empty dump is removed and counts as a failed optional step (`PARTIAL`). A manual run:

```
sudo ~/bin/backup-homelab.sh 2>&1 | tail -20
LAST=$(ls -1dt /var/backups/homelab/*/ | head -1); cat "$LAST/STATUS"; ls -lh "$LAST"
```

`status: OK`; the dump listed `portfolio.skills` (55 documents), `portfolio.projects` (9) and `portfolio.messages` (0); the archive is about 5 KB. The cron entry is unchanged, and the new function is installed with `sudo install -m 755 … ~/bin/backup-homelab.sh` as before.

## Verification

```
kubectl -n portfolio get cronjob ddns                          # last schedule within 5 minutes
kubectl -n monitoring get probe,prometheusrule                 # <DOMAIN_SLUG>-public, portfolio
kubectl -n portfolio get backendtrafficpolicy                  # portfolio-limits
cat "$(ls -1dt /var/backups/homelab/*/ | head -1)/STATUS"      # OK
```

## Limits

- **The monitor shares the house.** Prometheus, the probe and the alerts run in the cluster; if the power or the internet is down, nothing reports. The dead man's switch ([SRV-22](./SRV-22-Alerting-Reliability.md#limits)) has the same limit.
- **An address change still costs minutes.** Up to the cron interval plus the DNS TTL.
- **The updater depends on two outside calls:** Cloudflare's trace page for the address and the Cloudflare API for the record. If the trace page changes its format, the job fails and `PortfolioDdnsNotRunning` fires after 30 minutes.
- **The rate limit is per Envoy pod and per source address.** With one Envoy pod it is exact; visitors behind one shared address (an office, a mobile carrier) share one bucket. If the Cloudflare proxy were ever switched on, all visitors would arrive from Cloudflare's addresses and a `ClientTrafficPolicy` would be needed first.
- **A restore of the Mongo dump has not been tested.** The dump runs and has the right document counts; `mongorestore --archive --gzip` into an empty database is the untested half and should be done once, like the etcd restore test of [SRV-19](./SRV-19-Cluster-Upgrade.md).
- **No off-site copy, by decision.** All copies are in the house: the control plane, the worker mirror. A fire, theft or power surge that takes both loses the data.
- **The backup folder is sensitive.** The etcd snapshot holds every Secret in clear text, including the Cloudflare token and the SMTP password; the files are mode `600`.
- **Not yet recorded in the platform README.** The `cert-manager` and `blackbox` releases should get rows in `platform/README.md` (`helm list -A` shows the installed chart versions).

## Files

| File | Location | Contents |
|---|---|---|
| `ddns.yaml`, `monitoring.yaml`, `limits.yaml` | `apps/portfolio/` in `homelab-gitops` | Updater, probe and alerts, rate limit |
| `values.yaml` | `platform/blackbox-exporter/` | Exporter settings and the `public_https` module |
| `backup-homelab.sh` | `ops/backup/`; installed as `~/bin/backup-homelab.sh` | Nightly backup with the Mongo step |
| Secret `cloudflare-ddns` | `portfolio` namespace, not in Git | DNS API token and zone ID |
| `mongo-portfolio.archive.gz` | `/var/backups/homelab/<run>/` | The nightly database dump |

## Related

- [Portfolio Hosting](./SRV-24-Portfolio-Hosting.md) — what is being operated
- [Public Gateway and HTTPS](./SRV-23-Public-Gateway-and-HTTPS.md) — certificates and the Gateway the limits sit on
- [Alerting Reliability](./SRV-22-Alerting-Reliability.md) — the backup script and the dead man's switch
- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — Prometheus rules and the Slack route
- [SRV-25-TRBL](../troubleshooting/SRV-25-TRBL-Public-Site-Operations.md) — troubleshooting for this doc
- [Guide: Port Forwarding and a Changing Public IP](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/network/segmentation/Guide-Port-Forwarding-and-Dynamic-IP.md) — companion Guides repository
- [Guide: Logging and Alerting](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Logging-and-Alerting.md) — companion Guides repository
