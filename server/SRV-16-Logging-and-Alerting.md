---
tags: [homelab, project, kubernetes, logging, loki, alloy, alertmanager, slack, helm, grafana]
---

# Logging and Alerting

> Status: 🟢 **Working.** Loki stores the logs of the cluster's pods, Grafana Alloy collects them, Grafana can query them, and Alertmanager sends `warning` and `critical` alerts to a Slack channel (a test alert was delivered). 🟡 Open: no custom alert rules yet, none of the new data is backed up, and the laptop node is filtered out by label only — see [Open items](#open-items).

Picks up after [Persistent Storage](./SRV-15-Persistent-Storage.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Loki](#step-1--loki)
5. [Step 2 — Grafana Alloy](#step-2--grafana-alloy)
6. [Step 3 — Keep the laptop node out of log collection](#step-3--keep-the-laptop-node-out-of-log-collection)
7. [Step 4 — Loki as a Grafana data source](#step-4--loki-as-a-grafana-data-source)
8. [Step 5 — Slack webhook and Secret](#step-5--slack-webhook-and-secret)
9. [Step 6 — Alertmanager routing](#step-6--alertmanager-routing)
10. [Step 7 — Test alert](#step-7--test-alert)
11. [Verification](#verification)
12. [Limits](#limits)
13. [Open items](#open-items)
14. [Files](#files)
15. [Related](#related)

## Components

| Component | Installed as | Namespace | Role |
|---|---|---|---|
| Loki | Helm release `loki`, chart `grafana-community/loki` `18.15.1`, `deploymentMode: Monolithic` | `monitoring` | Stores and queries logs; one pod `loki-0` with a 10 GiB `local-path` volume |
| Grafana Alloy | Helm release `alloy`, chart `grafana/alloy` `1.13.1`, one-replica Deployment | `monitoring` | Reads the logs of all pods through the Kubernetes API and pushes them to Loki |
| Loki data source | `grafana.additionalDataSources` in `prometheus-values.yaml` | Grafana | Lets Grafana (Explore) query Loki |
| Alertmanager routing | `alertmanager.config` in `prometheus-values.yaml` | `monitoring` | Decides which alerts are sent where |
| Secret `alertmanager-slack` | Created by hand | `monitoring` | The Slack incoming-webhook URL |

Concepts (labels vs full-text index, the Alloy pipeline, how Alertmanager routes): [Guide: Logging and Alerting](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Logging-and-Alerting.md).

## Why

Before this step:

- A pod's log could only be read with `kubectl logs`, and it was gone once the pod was deleted or recreated, which is exactly when it is needed.
- Alertmanager ran with the chart's default configuration: it received alerts (including the always-firing `Watchdog` heartbeat) but had no receiver, so no alert ever left the cluster.

## Design decisions

| Decision | Reason |
|---|---|
| **Loki** for logs | Indexes only labels (namespace, pod, container), not log text, so it is small enough for a single node; Grafana already supports it |
| **Monolithic, one replica, filesystem storage** | Only one worker is schedulable. The chart's defaults (three replicas, S3-compatible storage) do not fit; with one replica `replication_factor` must be `1`, and the chart's bundled MinIO is deprecated and not used |
| **Grafana Alloy**, not Promtail | Promtail reached end of life on 2026-03-02; Alloy is Grafana's replacement collector |
| **Alloy as a Deployment, not a DaemonSet** | `kubectl cordon` does **not** keep DaemonSet pods away (the DaemonSet controller tolerates the unschedulable taint), and the laptop node must stay free of our services ([SRV-13](./SRV-13-ArgoCD-GitOps.md#open-items)). One Alloy pod reading logs through the API server avoids a pod on every node |
| **Slack** as the notification channel | Slack is the common chat tool for team alerting; Alertmanager supports it natively. Telegram was considered first and dropped for that reason |
| **Secret file, not inline value** | The webhook URL is a credential. It is mounted from a Secret and referenced with `api_url_file`, so it never appears in a values file or in git |

## Step 1 — Loki

The chart lives in the `grafana-community` repository (it moved there from `grafana`).

```
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
```

`~/k8s-manifests/loki-values.yaml`:

```yaml
deploymentMode: Monolithic

loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  compactor:
    retention_enabled: true
    delete_request_store: filesystem
  limits_config:
    retention_period: 168h

singleBinary:
  replicas: 1
  persistence:
    enabled: true
    storageClass: local-path
    size: 10Gi

backend:  { replicas: 0 }
read:     { replicas: 0 }
write:    { replicas: 0 }
gateway:  { enabled: false }
chunksCache:  { enabled: false }
resultsCache: { enabled: false }
lokiCanary:   { enabled: false }
test:         { enabled: false }
minio:        { enabled: false }
```

| Value | Effect |
|---|---|
| `deploymentMode: Monolithic` | All Loki roles in one pod (the name was `SingleBinary` before chart 12) |
| `auth_enabled: false` | Single tenant; no `X-Scope-OrgID` header needed |
| `replication_factor: 1` | Required with one replica |
| `schemaConfig` (tsdb, v13, `filesystem`) | The index and chunk format; the chart ships an empty one, so it has to be given |
| `compactor.retention_enabled` + `limits_config.retention_period: 168h` | Logs older than 7 days are deleted (retention only works with the compactor enabled) |
| the zero/disabled blocks | Remove everything a multi-replica install would add: separate read/write/backend pods, gateway, caches, canary, MinIO |

The values are checked by rendering before anything is installed, and the render must contain one StatefulSet and no gateway, canary or cache workloads:

```
helm template loki grafana-community/loki --version 18.15.1 \
  -n monitoring -f ~/k8s-manifests/loki-values.yaml > /tmp/loki-render.yaml \
  && echo OK && grep -E '^kind: (StatefulSet|Deployment|PersistentVolumeClaim)' /tmp/loki-render.yaml | sort | uniq -c
```

```
helm upgrade --install loki grafana-community/loki \
  --version 18.15.1 -n monitoring \
  -f ~/k8s-manifests/loki-values.yaml

kubectl -n monitoring get pods -o wide | grep loki
kubectl -n monitoring get pvc | grep loki
kubectl -n monitoring get svc | grep loki
```

Result: `loki-0` running, its claim `Bound` on `local-path`, and a Service `loki` on port 3100. The chart version is pinned for the same reason as in [SRV-15](./SRV-15-Persistent-Storage.md#step-2--grafana-admin-secret-and-volume): Helm must not pick up a newer chart as a side effect.

## Step 2 — Grafana Alloy

Alloy stays in the `grafana` repository. The latest chart version is read into a variable instead of being typed, then pinned:

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
ALLOY_VER=$(helm search repo grafana/alloy -o json | python3 -c 'import sys,json; print(json.load(sys.stdin)[0]["version"])')
echo "ALLOY_VER=$ALLOY_VER"
```

Result: `1.13.1`. The final `~/k8s-manifests/alloy-values.yaml` is in [Step 3](#step-3--keep-the-laptop-node-out-of-log-collection); the first install used the same file without the `drop` rule. Render check, then install:

```
helm template alloy grafana/alloy --version "$ALLOY_VER" -n monitoring \
  -f ~/k8s-manifests/alloy-values.yaml > /tmp/alloy-render.yaml \
  && echo OK && grep -E '^kind: (Deployment|DaemonSet)' /tmp/alloy-render.yaml

helm upgrade --install alloy grafana/alloy --version "$ALLOY_VER" -n monitoring \
  -f ~/k8s-manifests/alloy-values.yaml
```

The render must show a `Deployment` and no `DaemonSet`. The pod comes up as `2/2` (Alloy and its config reloader).

## Step 3 — Keep the laptop node out of log collection

Alloy's first logs showed repeated warnings: it tried to read the logs of pods on the laptop node and failed with `no route to host` on the kubelet port. The laptop is `NotReady` (the owner shuts it down), but system DaemonSet pods (kube-proxy, Flannel, MetalLB speaker, node-exporter) are still registered on it. Details: [SRV-16-TRBL](../troubleshooting/SRV-16-TRBL-Logging-and-Alerting.md#alloy-warnings-no-route-to-host-for-the-laptop-node).

The pods of that node are dropped before collection. Final `alloy-values.yaml`:

```yaml
controller:
  type: deployment
  replicas: 1

alloy:
  configMap:
    content: |
      discovery.kubernetes "pods" {
        role = "pod"
      }

      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets

        // laptop node (owned by another admin, often offline): do not collect its logs
        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          regex         = "<GPU_HOSTNAME>"
          action        = "drop"
        }
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          target_label  = "node"
        }
      }

      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"
        }
      }
```

The pipeline reads top to bottom: discover pods → drop the laptop's and turn pod metadata into labels → read each container's log → push to Loki. Applied with `helm upgrade alloy grafana/alloy --version 1.13.1 -n monitoring -f ~/k8s-manifests/alloy-values.yaml`. After the rollout, the warnings stopped.

## Step 4 — Loki as a Grafana data source

Added to the `grafana:` block of `~/k8s-manifests/prometheus-values.yaml`, so the data source is part of the release and survives a recreated pod:

```yaml
grafana:
  # ... admin and persistence blocks from SRV-15 ...
  additionalDataSources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki.monitoring.svc.cluster.local:3100
      isDefault: false
```

```
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --version 91.4.1 -n monitoring \
  -f ~/k8s-manifests/prometheus-values.yaml --dry-run > /dev/null && echo DRYRUN_OK

helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --version 91.4.1 -n monitoring \
  -f ~/k8s-manifests/prometheus-values.yaml

kubectl -n monitoring rollout restart deploy/prometheus-grafana
kubectl -n monitoring rollout status deploy/prometheus-grafana --timeout=180s
```

Grafana → Connections → Data sources now lists Alertmanager, Loki and Prometheus (Prometheus stays the default).

## Step 5 — Slack webhook and Secret

In Slack (browser):

1. Create a workspace and a channel `#alerts`.
2. `api.slack.com/apps` → **Create New App** → **Blank app** (this is the entry that used to be called "From scratch").
3. **Incoming Webhooks** → activate → **Add New Webhook to Workspace** → choose `#alerts`.
4. Copy the webhook URL. It is a credential: anyone with it can post to the channel.

The URL is entered without echo and stored in a Secret; it is never written to a file or to git:

```
read -rs -p "Slack webhook URL: " SLACK_URL; echo
```

```
kubectl -n monitoring create secret generic alertmanager-slack \
  --from-literal=webhook-url="$SLACK_URL"
unset SLACK_URL
kubectl -n monitoring get secret alertmanager-slack
```

`read` is run alone, before the rest: pasted in one block, it would take the following lines as its input.

## Step 6 — Alertmanager routing

Appended to `prometheus-values.yaml` as a new top-level `alertmanager:` block (backup taken first):

```yaml
alertmanager:
  alertmanagerSpec:
    # mounted at /etc/alertmanager/secrets/alertmanager-slack/
    secrets:
      - alertmanager-slack
  config:
    global:
      resolve_timeout: 5m
    templates:
      - /etc/alertmanager/config/*.tmpl
    route:
      receiver: "null"
      group_by: [alertname, namespace]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      routes:
        # internal heartbeat alerts, never notify
        - receiver: "null"
          matchers:
            - alertname =~ "Watchdog|InfoInhibitor"
        # laptop node (owned by another admin, often offline): ignore
        - receiver: "null"
          matchers:
            - node = "<GPU_HOSTNAME>"
        - receiver: "null"
          matchers:
            - instance =~ "<GPU_NODE_IP_REGEX>.*"
        - receiver: slack
          matchers:
            - severity =~ "warning|critical"
    inhibit_rules:
      - source_matchers:
          - severity = "critical"
        target_matchers:
          - severity =~ "warning|info"
        equal: [namespace, alertname]
    receivers:
      - name: "null"
      - name: slack
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/alertmanager-slack/webhook-url
            send_resolved: true
            title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}'
            text: |-
              {{ range .Alerts }}*{{ .Labels.severity }}* {{ .Annotations.summary }}
              {{ .Annotations.description }}
              {{ end }}
```

| Part | Effect |
|---|---|
| `alertmanagerSpec.secrets` | The operator mounts the Secret into the Alertmanager pod under `/etc/alertmanager/secrets/<name>/` |
| `route` (first match wins) | Heartbeat alerts and anything about the laptop node go to the `"null"` receiver (dropped); `warning` and `critical` go to Slack; everything else, such as `info`, goes to `"null"` by default |
| `group_by`, `group_wait`, `group_interval`, `repeat_interval` | One message per alert name and namespace, first sent after 30 s, updates every 5 min, reminder every 12 h |
| `inhibit_rules` | A `critical` alert silences the `warning`/`info` alert of the same name and namespace |
| `send_resolved: true` | A `[RESOLVED]` message follows when the alert clears |

The laptop matchers are written as the node name (`node` label, set by kube-state-metrics alerts) and an escaped-dot regex of its IP (`instance` label, set by node-exporter alerts).

```
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --version 91.4.1 -n monitoring \
  -f ~/k8s-manifests/prometheus-values.yaml --dry-run > /dev/null && echo DRYRUN_OK

helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --version 91.4.1 -n monitoring \
  -f ~/k8s-manifests/prometheus-values.yaml

kubectl -n monitoring get pods | grep alertmanager
kubectl -n monitoring logs alertmanager-prometheus-kube-prometheus-alertmanager-0 -c alertmanager --tail=15
```

The Alertmanager pod restarts and runs `2/2`; a successful load shows `Loading configuration file` in the log and no `error`.

## Step 7 — Test alert

A synthetic alert is sent from inside the Alertmanager pod with `amtool`:

```
kubectl -n monitoring exec alertmanager-prometheus-kube-prometheus-alertmanager-0 -c alertmanager -- \
  amtool alert add TestAlert severity=warning namespace=test \
  --annotation=summary="Test alert from homelab" \
  --annotation=description="If you see this in Slack, the pipeline works." \
  --alertmanager.url=http://localhost:9093
```

After the 30 s `group_wait`, a `[FIRING:1] TestAlert` message appeared in `#alerts`, and a `[RESOLVED]` message followed when the test alert expired.

## Verification

| Check | Result |
|---|---|
| `kubectl -n monitoring get pods \| grep loki` | `loki-0` running |
| `kubectl -n monitoring get pvc \| grep loki` | Claim `Bound` on `local-path` |
| `kubectl -n monitoring get pods -o wide \| grep alloy` | `2/2 Running` |
| Alloy log after Step 3 | No `no route to host` warnings |
| Grafana → Data sources | Alertmanager, Loki, Prometheus listed |
| Grafana → Explore → Loki, query `{namespace="argocd"}` | ArgoCD log lines shown |
| `amtool alert add TestAlert …` | `[FIRING:1] TestAlert` and later `[RESOLVED]` in Slack |

Useful queries in Grafana Explore (Loki, **Code** mode):

```
{namespace="monitoring", container="alertmanager"}
{namespace="monitoring", container="alertmanager"} |= "notify"
{namespace="monitoring", container="alertmanager"} |= "level=error"
```

Alertmanager logs only failed notifications; a delivered message leaves no log line, so an empty result with a message in Slack is the normal case.

## Limits

- **Loki's data is a directory on one node's disk,** like every other `local-path` volume ([SRV-15](./SRV-15-Persistent-Storage.md#limits-of-this-storage)): not a backup, not replicated. Retention is set to 7 days; the deletion itself has not been observed yet.
- **One Alloy pod** is a single point of failure for collection. It has no volume, so reading positions are not kept across a restart (not verified); lines written while it is down may be missed. It reads through the API server rather than from the node's disk, which adds some load to the API server; at this cluster size that is acceptable.
- **Alertmanager has no volume,** so its silences are lost on restart.
- **The laptop node is filtered by label.** An alert that carries neither a `node` nor an `instance` label of that node (a `TargetDown` for the node-exporter job, for example) is not covered by the rules above.
- **The Slack webhook lives only in the cluster Secret.** If it is lost, a new one is created in Slack and the Secret is replaced.

## Open items

- ⬜ **Alert for the own workload,** for example "`my-site` has no ready pod for 5 minutes", broken on purpose to see the message arrive — so far only the chart's built-in rules exist.
- ⬜ **Check the laptop filter with the laptop offline:** wait for a full alert evaluation cycle and confirm nothing about that node reaches Slack.
- ⬜ **Confirm retention:** after more than 7 days, check that old log chunks are deleted and the volume does not fill.
- ⬜ **Volumes for Alertmanager and no backup for Loki/Grafana/Prometheus** (continuing [SRV-15](./SRV-15-Persistent-Storage.md#open-items)).
- ⬜ **Helm releases are still applied by hand,** values in `~/k8s-manifests/`, not through ArgoCD.
- ⬜ **Optional second receiver** (an on-call tool such as PagerDuty) once there is a reason for paging.

## Files

| File | Location | Contents |
|---|---|---|
| `loki-values.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | Loki: monolithic, filesystem storage, 7-day retention, 10 GiB volume |
| `alloy-values.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | Alloy: Deployment and the log pipeline, laptop node dropped |
| `prometheus-values.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | Extended: Loki data source and Alertmanager routing (besides the SRV-15 content) |
| Secret `alertmanager-slack` | Namespace `monitoring` | The Slack webhook URL; created by hand, never committed |

## Related

- [Persistent Storage](./SRV-15-Persistent-Storage.md) — the previous step; the volumes used here
- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — where Prometheus, Grafana and Alertmanager were installed
- [SRV-16-TRBL](../troubleshooting/SRV-16-TRBL-Logging-and-Alerting.md) — problems hit during this step
- [Guide: Logging and Alerting](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Logging-and-Alerting.md) — companion Guides repository
- [Guide: Helm Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Helm-Basics.md) — companion Guides repository
