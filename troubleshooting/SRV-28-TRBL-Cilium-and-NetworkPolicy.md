---
tags: [homelab, project, kubernetes, cilium, networkpolicy, troubleshooting]
---

# Troubleshooting — Cilium and NetworkPolicy

> Companion to [Cilium CNI Migration and NetworkPolicy](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md), which records only the working procedure. This file records the problems hit during that work. None of them caused an outage that outlived the migration.

## Contents

| Incident | Step |
|---|---|
| [Backup script fails with `permission denied`](#backup-script-fails-with-permission-denied) | [Step 2](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-2--backup) |
| [Kafka exporter and entity operator crash after the worker drain](#kafka-exporter-and-entity-operator-crash-after-the-worker-drain) | [Step 4](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-4--migrate-the-nodes) |
| [Test pod shows no output on a fresh node](#test-pod-shows-no-output-on-a-fresh-node) | [Step 4](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-4--migrate-the-nodes) |
| [Reboot of the laptop is inhibited](#reboot-of-the-laptop-is-inhibited) | [Step 4](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-4--migrate-the-nodes) |
| [CoreDNS would go down with the control-plane drain](#coredns-would-go-down-with-the-control-plane-drain) | [Step 4](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-4--migrate-the-nodes) |
| [MetalLB `statuscleaner` in `CrashLoopBackOff`](#metallb-statuscleaner-in-crashloopbackoff) | [Step 5](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-5--finish-and-remove-flannel) |

## Backup script fails with `permission denied`

**Symptom:** the backup run before the migration ended with `status: FAIL`, and Slack reported it:

```
error: error loading config file "/etc/kubernetes/admin.conf": open /etc/kubernetes/admin.conf: permission denied
tar: pki/ca.key: Cannot open: Permission denied
status: FAIL, backup in /var/backups/homelab/<timestamp>-FAILED
```

Only the `manifests` step was `ok`.

**Root cause:** the script was started as the normal user. It reads the admin kubeconfig and the private keys under `pki`, both readable only by root. The nightly cron entry runs as root, so it never showed this.

**Fix:** `sudo ~/bin/backup-homelab.sh`. The result was `status: OK` with five `ok` parts. The `-FAILED` folder of the first run was removed by hand; the script would have removed it after seven days anyway. The failed run also proved that the failure report from [SRV-22](../server/SRV-22-Alerting-Reliability.md#step-4--resilient-backup-script) reaches Slack.

## Kafka exporter and entity operator crash after the worker drain

**Symptom:** right after the worker came back, `homelab-kafka-exporter` was in `CrashLoopBackOff` and the entity operator restarted repeatedly; the broker pod was not in the list at all:

```
Error Init Kafka Client: kafka: client has run out of available brokers to talk to: dial tcp <service IP>:9091: connect: connection refused
```

**Root cause:** not the network. The drain evicted the Strimzi cluster operator together with the broker. The broker is created by the operator, which had to start first, win the leader election and finish its first reconciliation of the `Kafka` object. Until the broker pod existed, everything that talks to it was refused. The `Ready: True` shown by `kubectl get kafka` at that moment was the old status from before the drain.

**Fix:** waiting. About nine minutes after the worker came back the broker was `1/1`, and the exporter and entity operator settled on their own. The consumer group `alloy-consumer` showed `LAG 0` to `3` afterwards, so the pipeline caught up. While the broker is missing, logs are not written to Loki; how much a collector can hold meanwhile is the limit already recorded in [SRV-21](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#limits). Rule for the next time: after draining the worker, watch `kubectl -n kafka get pods` for the broker before touching the next node.

## Test pod shows no output on a fresh node

**Symptom:** the test pod on the laptop printed only `If you don't see a command prompt, try pressing enter.` and was deleted; no address, no health check.

**Root cause:** `kubectl run --attach --rm` attaches as soon as the container starts. On a node that has not pulled the image yet the pull takes a while, and the short-lived command finishes before the attach is established, so its output is lost.

**Fix:** run the pod without `--attach`, wait for the phase `Succeeded`, then read `kubectl logs`, then delete it (the form used in [Step 4](../server/SRV-28-Cilium-CNI-Migration-and-NetworkPolicy.md#step-4--migrate-the-nodes)). The output then showed `10.245.1.20` and `ok`.

## Reboot of the laptop is inhibited

**Symptom:**

```
Operation inhibited by "<GPU_USER>" (PID ... "gnome-session-s", user <GPU_USER>), reason is "user session inhibited".
User <GPU_USER> is logged in on sshd.
User <GPU_USER> is logged in on tty2.
```

**Root cause:** the laptop runs a desktop session, and systemd refuses a plain reboot while a graphical session is open.

**Fix:** `sudo systemctl reboot -i` ignores the inhibitors. The desktop session is closed without saving, so the reboot is done when nothing is open on the laptop; the person has to log in again afterwards. The node returned `Ready` within about two minutes.

## CoreDNS would go down with the control-plane drain

**Symptom:** none yet; found by listing the pods on the control plane before the drain. Both CoreDNS pods ran there, still on the old network (`10.244.0.x`).

**Root cause:** a drain evicts them at once; cluster DNS is unavailable until the replacements start on other nodes, and everything that resolves names during that time fails.

**Fix:** cordon the node first, then `kubectl -n kube-system rollout restart deploy/coredns`. With the node closed to scheduling the new pods land on the already migrated nodes (`10.245.x.x`), the old ones terminate after the new are ready, and the drain that follows has nothing important left to evict.

## MetalLB `statuscleaner` in `CrashLoopBackOff`

**Symptom:** after the control-plane reboot, one pod stayed in `CrashLoopBackOff` for nine minutes:

```
metallb-frr-k8s-statuscleaner-...   0/1   CrashLoopBackOff
unable to start manager ... Get "https://<service IP>:443/api": dial tcp <service IP>:443: connect: connection refused
```

**Root cause:** the pod was evicted from the control plane by the drain and started on the worker while the API server was down for its reboot. It uses the node's network and exited when it could not reach the API; Kubernetes then waits longer between restarts (up to five minutes), so it stayed down after the API returned. Not related to Cilium.

**Fix:** `kubectl -n metallb-system rollout restart deploy/metallb-frr-k8s-statuscleaner`; the new pod was `1/1` after twenty seconds. The other MetalLB pods (controller, speakers, FRR) were unaffected.
