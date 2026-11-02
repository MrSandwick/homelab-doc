---
tags: [homelab, project, kubernetes, ddns, prometheus, blackbox-exporter, argocd, troubleshooting]
---

# Troubleshooting — Public Site Operations

> Companion to [Public Site Operations](../server/SRV-25-Public-Site-Operations.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [`cronjob "ddns" not found` right after the merge](#cronjob-ddns-not-found-right-after-the-merge) | [Step 1](../server/SRV-25-Public-Site-Operations.md#step-1--keep-the-dns-record-on-the-current-address) |
| [A test job that printed nothing](#a-test-job-that-printed-nothing) | [Step 1](../server/SRV-25-Public-Site-Operations.md#step-1--keep-the-dns-record-on-the-current-address) |
| [No `wget` in the Prometheus container](#no-wget-in-the-prometheus-container) | [Step 2](../server/SRV-25-Public-Site-Operations.md#step-2--probe-and-alerts) |
| [New probe and rules not visible in Prometheus](#new-probe-and-rules-not-visible-in-prometheus) | [Step 2](../server/SRV-25-Public-Site-Operations.md#step-2--probe-and-alerts) |

## `cronjob "ddns" not found` right after the merge

**Symptom:** `kubectl create job --from=cronjob/ddns ddns-test` answered `cronjobs.batch "ddns" not found` minutes after the pull request was created.

**Root cause:** ArgoCD polls Git about every three minutes; the pull request had not been merged yet, or the sync had not run.

**Fix:** `git fetch && git log --oneline origin/main -3` showed whether the commit was on `main`, `kubectl -n argocd get application portfolio` whether it was synced; a hard refresh (`kubectl -n argocd annotate application portfolio argocd.argoproj.io/refresh=hard --overwrite`) forces a re-read. The CronJob appeared once the merge and the sync were done.

## A test job that printed nothing

**Symptom:** `kubectl logs job/ddns-test` after `sleep 15` returned no lines.

**Root cause:** the first run had to pull the `curlimages/curl` image; the container had not started within 15 seconds.

**Fix:** the same `kubectl -n portfolio logs job/ddns-test` a little later answered `unchanged: <PUBLIC_IP>`.

## No `wget` in the Prometheus container

**Symptom:** `kubectl -n monitoring exec prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- wget …` failed with `exec: "wget": executable file not found in $PATH`.

**Root cause:** the Prometheus image is minimal and carries neither `wget` nor `curl`.

**Fix:** query the HTTP API from the control plane through the Service address instead:

```
PIP=$(kubectl -n monitoring get svc prometheus-kube-prometheus-prometheus -o jsonpath='{.spec.clusterIP}')
curl -s "http://$PIP:9090/api/v1/rules?type=alert"
```

## New probe and rules not visible in Prometheus

**Symptom:** a few seconds after ArgoCD created the `Probe` and the `PrometheusRule`, the rules query returned nothing and `probe_success` returned an empty result.

**What was checked:** the selectors of the Prometheus object (`kubectl -n monitoring get prometheus -o jsonpath=…`): both `probeSelector` and `ruleSelector` are `release=prometheus`, the label already on `my-site-alerts` and on the new objects, so the objects were selected. Only the age of the objects was wrong: they were about a minute old.

**Root cause:** the operator reloads Prometheus a minute or two after new objects appear, and the first probe sample comes with the next 60-second cycle.

**Fix:** wait, with a loop that stops when the three rules are loaded:

```
for i in $(seq 1 12); do n=$(curl -s "http://$PIP:9090/api/v1/rules?type=alert" | grep -o '"name":"Portfolio[A-Za-z]*"' | sort -u | wc -l); echo "attempt $i: $n of 3 rules loaded"; [ "$n" = 3 ] && break; sleep 15; done
```

The rules were loaded on the first attempt of the second try, and `probe_success` followed.

## Related

- [Public Site Operations](../server/SRV-25-Public-Site-Operations.md) — the working procedure
- [SRV-16-TRBL](./SRV-16-TRBL-Logging-and-Alerting.md) — earlier alerting incidents
- [SRV-24-TRBL](./SRV-24-TRBL-Portfolio-Hosting.md) — ArgoCD behaviour for the same Application
