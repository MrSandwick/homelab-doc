---
tags: [homelab, project, kubernetes, alerting, alertmanager, prometheus, slack, backup, systemd, dead-mans-switch]
---

# Alerting Reliability

> Status: 🟢 **Working.** A dead man's switch on the control plane reports to Slack when the API server, a node or the alerting pipeline itself is down; Slack alerts now name the cluster and say what and where; the nightly backup no longer stops when the worker is unreachable and reports partial or failed runs. All three were tested. 🟡 Limit: nothing reports if the control plane itself, the power or the home internet is down — see [Limits](#limits).

Picks up after [Kafka with Strimzi and a Log Pipeline](./SRV-21-Kafka-Strimzi-Log-Pipeline.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the laptop node `<GPU_HOSTNAME>`, the admin user `<USER>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Dead man's switch](#step-1--dead-mans-switch)
5. [Step 2 — Test the switch](#step-2--test-the-switch)
6. [Step 3 — Readable Slack messages](#step-3--readable-slack-messages)
7. [Step 4 — Resilient backup script](#step-4--resilient-backup-script)
8. [Verification](#verification)
9. [Limits](#limits)
10. [Files](#files)
11. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| `homelab-deadman.sh` + systemd timer (every 5 min) | Control plane, `/usr/local/bin/` and `/etc/systemd/system/` | Checks the API server, node readiness and the `Watchdog` alert; posts to Slack directly |
| `/etc/homelab-deadman/slack-url` | Control plane, root only (`600`) | Copy of the Slack webhook URL, so the switch can report even when the API server cannot be asked for the Secret |
| `externalLabels.cluster: homelab` | Prometheus (`kube-prometheus-stack` values) | Adds the cluster name to every series and alert |
| `slack.tmpl` | Alertmanager (`alertmanager.templateFiles`) | Message layout: status, cluster, alert, summary, severity, location, start time |
| `backup-homelab.sh` (rewritten) | Control plane, `~/bin/`, run by the existing cron entry | Critical and optional steps, a `STATUS` file per run, safe rotation, Slack report on problems |

## Why

In late October the worker was found switched off after more than two days, and nothing had reported it. The reason is structural: Prometheus and Alertmanager run on the worker ([SRV-17](./SRV-17-Dedicated-GPU-Node.md#node-roles)), so when the worker goes down, the system that should raise the alarm goes down with it. The same outage left the backups half-done: the Grafana step timed out, the script stopped, and rotation and the mirror never ran — silently.

Three changes follow from that incident:

- something outside the worker has to notice that alerting is gone (**dead man's switch**);
- messages have to be readable at a glance and name the cluster, because there may be more than one;
- the backup must degrade instead of aborting, and say so.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Watcher location | The control plane | Always on, and a different node from Prometheus and Alertmanager; nothing to install |
| External heartbeat service (Healthchecks.io, Dead Man's Snitch) | Not used | Decided to keep everything inside the lab. The standard pattern is still the same: watch the always-firing `Watchdog` from outside the pipeline. The cost is the limit in [Limits](#limits) |
| What is checked | API server `/readyz`; every node except the laptop is `Ready`; `Watchdog` active in Alertmanager | `Watchdog` reaching Alertmanager proves Prometheus evaluates rules and Alertmanager receives them — the whole pipeline in one check |
| How Alertmanager is asked | Through the API server's service proxy (`/api/v1/namespaces/monitoring/services/…:9093/proxy/api/v2/alerts`) | No port-forward, no exposed port; uses the admin kubeconfig the node already has |
| Notification path | Slack webhook directly from the script | Must not depend on Alertmanager, which is one of the things being checked |
| Noise | Message on change, reminder every hour, one recovery message | A failure that lasts all night produces a handful of messages, not hundreds |
| Interval | 5 minutes | Enough for a home lab; worst-case detection about 5 minutes |
| Cluster name | Prometheus `externalLabels` | Attached at the source, so every alert carries it without changing any rule |
| Backup: what is critical | etcd snapshot, certificates, manifests | These are what a restore needs; Grafana data is convenient but replaceable |
| Backup: failed runs | Folder renamed `*-FAILED`, excluded from rotation, removed after 7 days | A broken backup can never push a good one out of the last seven |

## Step 1 — Dead man's switch

The webhook URL is copied from the existing Secret into a root-only file:

```
sudo mkdir -p /etc/homelab-deadman /var/lib/homelab-deadman
kubectl -n monitoring get secret alertmanager-slack -o jsonpath='{.data.webhook-url}' \
  | base64 -d | sudo tee /etc/homelab-deadman/slack-url > /dev/null
sudo chmod 600 /etc/homelab-deadman/slack-url
```

`ops/deadman/homelab-deadman.sh` in `homelab-gitops`, installed with `sudo install -m 755 … /usr/local/bin/`:

```
#!/usr/bin/env bash
set -uo pipefail

CLUSTER="homelab"
IGNORE_NODES="<GPU_HOSTNAME>"                # laptop node, often offline by design
REMIND_EVERY=3600                            # repeat a failure message every hour
export KUBECONFIG=/etc/kubernetes/admin.conf
SLACK_URL_FILE=/etc/homelab-deadman/slack-url
STATE_FILE=/var/lib/homelab-deadman/state
AM_PATH="/api/v1/namespaces/monitoring/services/prometheus-kube-prometheus-alertmanager:9093/proxy/api/v2/alerts?filter=alertname%3D%22Watchdog%22"

problems=()

# 1. API server
if ! kubectl get --raw /readyz > /dev/null 2>&1; then
  problems+=("Kubernetes API server is not answering on $(hostname)")
else
  # 2. nodes that are not Ready (except ignored ones)
  notready=$(kubectl get nodes --no-headers | awk -v ign="$IGNORE_NODES" '$1 != ign && $2 != "Ready" {print $1 " (" $2 ")"}')
  [ -n "$notready" ] && problems+=("Node not Ready: $notready")

  # 3. Watchdog must be active in Alertmanager, otherwise alerting itself is down
  if ! kubectl get --raw "$AM_PATH" 2>/dev/null | grep -q '"alertname":"Watchdog"'; then
    problems+=("Alerting pipeline is down: Watchdog is not active in Alertmanager ...")
  fi
fi

# send_slack: JSON built with python3 (safe quoting), posted with curl -m 10
# state file "OK|FAIL <last sent>": message on change, reminder every hour, recovery message
```

The full script, with the state handling, is in the repository. The timer:

```
# /etc/systemd/system/homelab-deadman.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/homelab-deadman.sh

# /etc/systemd/system/homelab-deadman.timer
[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now homelab-deadman.timer
systemctl list-timers homelab-deadman.timer --no-pager
sudo /usr/local/bin/homelab-deadman.sh      # prints "OK: API, nodes and Watchdog are fine"
```

Copies of both unit files are kept in `ops/deadman/`.

## Step 2 — Test the switch

Alertmanager is stopped through its custom resource (the operator removes the pod), the check is run by hand instead of waiting for the timer, then everything is restored:

```
kubectl -n monitoring patch alertmanager prometheus-kube-prometheus-alertmanager \
  --type merge -p '{"spec":{"replicas":0}}'
kubectl -n monitoring wait --for=delete pod/alertmanager-prometheus-kube-prometheus-alertmanager-0 --timeout=120s
sudo /usr/local/bin/homelab-deadman.sh      # red message in Slack

kubectl -n monitoring patch alertmanager prometheus-kube-prometheus-alertmanager \
  --type merge -p '{"spec":{"replicas":1}}'
kubectl -n monitoring wait --for=condition=Ready pod/alertmanager-prometheus-kube-prometheus-alertmanager-0 --timeout=180s
sleep 60                                     # Prometheus re-sends Watchdog about once a minute
sudo /usr/local/bin/homelab-deadman.sh      # green recovery message in Slack
```

Both messages arrived in Slack.

## Step 3 — Readable Slack messages

Three changes in `platform/kube-prometheus-stack/values.yaml`:

```
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: homelab

alertmanager:
  templateFiles:
    slack.tmpl: |-
      {{ define "slack.homelab.title" -}}
      {{ if eq .Status "firing" }}{{ if eq .CommonLabels.severity "critical" }}🔴{{ else }}🟠{{ end }}{{ else }}✅{{ end }} [{{ or .CommonLabels.cluster "unknown-cluster" }}] {{ .CommonLabels.alertname }}{{ if gt (len .Alerts) 1 }} ({{ len .Alerts }}){{ end }}{{ if eq .Status "resolved" }} — resolved{{ end }}
      {{- end }}

      {{ define "slack.homelab.text" -}}
      {{ range .Alerts -}}
      *{{ if .Annotations.summary }}{{ .Annotations.summary }}{{ else }}{{ .Labels.alertname }}{{ end }}*
      {{ if .Annotations.description }}{{ .Annotations.description }}
      {{ end -}}
      • Severity: `{{ .Labels.severity }}`{{ if .Labels.namespace }}  • Namespace: `{{ .Labels.namespace }}`{{ end }}{{ if .Labels.pod }}  • Pod: `{{ .Labels.pod }}`{{ end }}{{ if .Labels.node }}  • Node: `{{ .Labels.node }}`{{ end }}
      • Since: {{ .StartsAt.Format "2006-01-02 15:04 MST" }}{{ if eq .Status "resolved" }}  • Resolved: {{ .EndsAt.Format "2006-01-02 15:04 MST" }}{{ end }}
      {{ if .Annotations.runbook_url }}• Runbook: {{ .Annotations.runbook_url }}
      {{ end }}
      {{ end -}}
      {{- end }}
  config:
    receivers:
      - name: slack
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/alertmanager-slack/webhook-url
            send_resolved: true
            color: '{{ if eq .Status "firing" }}{{ if eq .CommonLabels.severity "critical" }}danger{{ else }}warning{{ end }}{{ else }}good{{ end }}'
            title: '{{ template "slack.homelab.title" . }}'
            text: '{{ template "slack.homelab.text" . }}'
```

The chart mounts `templateFiles` under `/etc/alertmanager/config/`, which the existing `templates: [/etc/alertmanager/config/*.tmpl]` line already loads. Applied with the installed chart version:

```
helm upgrade prometheus prometheus-community/kube-prometheus-stack -n monitoring \
  --version <installed chart version> -f platform/kube-prometheus-stack/values.yaml
```

Test alert, which expires after a minute so the resolved message follows:

```
kubectl -n monitoring exec alertmanager-prometheus-kube-prometheus-alertmanager-0 -c alertmanager -- \
  amtool alert add TestReadableAlert \
    severity=warning cluster=homelab namespace=default pod=my-site-test node=<WORKER_HOSTNAME> \
    --annotation=summary="Test: readable Slack message" \
    --annotation=description="If this message shows cluster, severity and pod, the template works." \
    --end="$(date -u -d '+1 minute' +%Y-%m-%dT%H:%M:%SZ)" \
    --alertmanager.url=http://localhost:9093
```

Slack showed `🟠 [homelab] TestReadableAlert` with an orange bar, the summary, the description and the line `Severity / Namespace / Pod / Node / Since`, then `✅ [homelab] TestReadableAlert — resolved` in green.

## Step 4 — Resilient backup script

The script from [SRV-18](./SRV-18-Backups-and-Platform-Config.md#step-1--the-backup-script) ran with `set -e`: any failing command stopped it. Now each part is a function run by a `step` wrapper that records failures instead of exiting:

```
step critical "etcd snapshot"        backup_etcd
step critical "certificates (pki)"   backup_pki
step critical "manifests"            backup_manifests
step optional "Grafana data"         backup_grafana
```

| Outcome | `STATUS` | Folder | Slack |
|---|---|---|---|
| All steps succeed | `OK` | kept | nothing |
| Only Grafana fails (worker down) | `PARTIAL` | kept, counts as a backup | orange message |
| A critical step fails | `FAIL` | renamed `*-FAILED`, removed after 7 days | red message |

Other details:

- The temporary `grafana-backup` pod is deleted in every case, also after a timeout; a partial `grafana-data.tar.gz` is removed.
- Rotation runs in every case and keeps the last seven folders that are not `-FAILED`.
- The PVC name and the wait timeout can be overridden from the environment (`GRAFANA_PVC`, `GRAFANA_TIMEOUT`), which makes the failure path testable in seconds.
- The script exits non-zero only on `FAIL`.

The source is `ops/backup/backup-homelab.sh` in `homelab-gitops`, installed over the old file with `install -m 755 … ~/bin/backup-homelab.sh`; the cron entries are unchanged.

Test:

```
sudo /home/<USER>/bin/backup-homelab.sh                                   # status: OK
sudo GRAFANA_PVC=does-not-exist GRAFANA_TIMEOUT=10s /home/<USER>/bin/backup-homelab.sh   # status: PARTIAL
for d in /var/backups/homelab/*/; do echo "$(basename $d)  $(cat $d/STATUS 2>/dev/null || echo old)"; done
kubectl -n monitoring get pod grafana-backup                              # NotFound
```

The first run reported `OK` with no message, the second `PARTIAL` with an orange message in Slack, at most seven folders remained, and no backup pod was left behind.

## Verification

```
systemctl list-timers homelab-deadman.timer --no-pager
sudo cat /var/lib/homelab-deadman/state                 # OK <timestamp>
cat /var/backups/homelab/$(ls -1t /var/backups/homelab | head -1)/STATUS
```

The timer has a next run within five minutes, the state is `OK`, the latest backup is `OK`.

## Limits

- **The watcher shares the house.** If the control plane, the power or the internet connection is down, nothing reports. Only an external heartbeat service closes that gap.
- **The switch checks presence, not correctness.** A rule that is wrong or missing is not detected; only a stopped pipeline, an unreachable API or a node that is not `Ready`.
- **The laptop node is excluded** from the node check by name, the same way it is excluded from alert routing ([SRV-16](./SRV-16-Logging-and-Alerting.md)).
- **The mirror to the worker is not checked.** The `rsync` cron entry still runs without a report; a worker that is down is reported by the switch instead.
- **Slack is a single channel.** If Slack or the webhook is broken, both the alerts and the switch are silent.

## Files

| File | Location | Contents |
|---|---|---|
| `homelab-deadman.sh`, `homelab-deadman.service`, `homelab-deadman.timer` | `ops/deadman/` in `homelab-gitops`; installed in `/usr/local/bin/` and `/etc/systemd/system/` | Dead man's switch |
| `backup-homelab.sh` | `ops/backup/`; installed as `~/bin/backup-homelab.sh` | Nightly backup |
| `values.yaml` | `platform/kube-prometheus-stack/` | `externalLabels`, `templateFiles`, Slack receiver fields |
| `slack-url` | `/etc/homelab-deadman/` on the control plane, not in Git | Webhook URL for the switch and the backup report |
| `state` | `/var/lib/homelab-deadman/` | Last state and time of the last message |

## Related

- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — Alertmanager, the Slack route and the `Watchdog` alert
- [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md) — the first version of the backup script and the cron entries
- [Dedicated GPU Node](./SRV-17-Dedicated-GPU-Node.md) — why everything runs on one worker
- [Guide: Logging and Alerting](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Logging-and-Alerting.md) — companion Guides repository
- [PromLabs: End-to-End Watchdog Alerts](https://training.promlabs.com/training/monitoring-and-debugging-prometheus/metrics-based-meta-monitoring/end-to-end-watchdog-alerts/) — the watchdog pattern
