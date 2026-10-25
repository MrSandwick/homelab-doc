---
tags: [homelab, project, kubernetes, kafka, alloy, loki, troubleshooting]
---

# Troubleshooting — Kafka Log Pipeline

> Companion to [Kafka with Strimzi and a Log Pipeline](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md), which records only the working procedure. This file records the problems hit during that work.

## Contents

| Incident | Step |
|---|---|
| [Consumer crash loop: `unrecognized attribute name "topic"`](#consumer-crash-loop-unrecognized-attribute-name-topic) | [Step 6](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#step-6--consumer-alloy) |
| [Logs arrive under `unknown_service` without labels](#logs-arrive-under-unknown_service-without-labels) | [Step 6](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#step-6--consumer-alloy) |
| [Kafka does not hold the backlog while Loki is down](#kafka-does-not-hold-the-backlog-while-loki-is-down) | [Step 8](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#step-8--loki-outage-test) |
| [The per-minute count shows a dip that is not a loss](#the-per-minute-count-shows-a-dip-that-is-not-a-loss) | [Step 8](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#step-8--loki-outage-test) |
| [`400 timestamp too old` from Loki](#400-timestamp-too-old-from-loki) | [Step 8](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#step-8--loki-outage-test) |

## Consumer crash loop: `unrecognized attribute name "topic"`

**Symptom:** the first `alloy-kafka` pod stays `1/2 CrashLoopBackOff`:

```
Error: /etc/alloy/config.alloy:5:3: unrecognized attribute name "topic"
```

After the values were fixed and `helm upgrade` ran, the same pod kept crashing with the same message.

**Root cause:** two separate things.

- In Alloy v1.20, `otelcol.receiver.kafka` takes the topic list and the encoding inside a `logs { topics = [...], encoding = "..." }` block. The top-level `topic` from older examples is gone.
- The ConfigMap was updated, but a container in `CrashLoopBackOff` waits out a growing back-off before the next start, and the logs shown were from the old attempt.

**Fix:** move the settings into the `logs` block (checked against the component reference for the installed version), then `kubectl -n monitoring rollout restart deploy/alloy-kafka` to replace the pod immediately. Verify the ConfigMap content first (`kubectl -n monitoring get configmap alloy-kafka -o yaml`), then read the logs of the newest pod, not of the one being terminated.

## Logs arrive under `unknown_service` without labels

**Symptom:** with the consumer exporting through `otelcol.exporter.otlphttp` to Loki's `/otlp` endpoint (the layout of Grafana's example), the topic grew and the consumer lag stayed near `0`, but `{namespace="monitoring"}` returned nothing for new data. The series API showed a single stream:

```
{"service_name":"unknown_service"}
```

The problem was hidden at first because the producer still pushed directly to Loki as well; the old streams carried the expected labels.

**Root cause:** `otelcol.receiver.loki` turns Loki labels into OTLP **log attributes**. Loki's OTLP endpoint indexes only selected **resource** attributes as labels and stores log attributes as structured metadata, so `namespace`, `pod`, `container`, `node` were no longer labels and the stream had no `service.name`.

**Fix:** convert back to Loki's push format in the consumer and name the attributes that must become labels:

```
otelcol.processor.attributes "labels" {
  action {
    key    = "loki.attribute.labels"
    action = "insert"
    value  = "namespace, pod, container, node, job, instance"
  }
  ...
}
otelcol.exporter.loki "default" { forward_to = [loki.write.default.receiver] }
```

The label names are the same as before; existing dashboards and queries work unchanged. The alternative — moving the attributes to resource attributes and configuring Loki's `otlp_config` — would rename or re-plumb the labels and needs a Loki change.

**Lesson:** when switching a data path, check the result with the old path switched **off**; while both write, the old one masks the new one.

## Kafka does not hold the backlog while Loki is down

**Symptom:** Loki was scaled to `0`. Four minutes later the group `alloy-consumer` still showed `LAG 0` on every partition, the same as with the first configuration.

**Root cause:** by default `otelcol.receiver.kafka` marks a message as read as soon as it is fetched, before it is forwarded. And the rest of the chain is asynchronous: the batch processor and `loki.write` accept records into memory and return at once. With the default `max_backoff_retries = 10`, `loki.write` gives up after about eight and a half minutes of retries and drops the batch.

**Fix:** `message_marking { after_execution = true }` on the receiver, and `max_backoff_retries = 0` (retry forever) on `loki.write`. With that, a five-minute outage loses nothing. `LAG` still stays at `0`, because `loki.write` accepts the records and holds them in memory; that remaining gap is described in [Limits](../server/SRV-21-Kafka-Strimzi-Log-Pipeline.md#limits). Making Kafka itself hold the backlog needs a synchronous last hop that returns the error to the receiver.

## The per-minute count shows a dip that is not a loss

**Symptom:** `sum(count_over_time({namespace=~".+"}[1m]))` per minute showed about 890 and 50 lines during the outage against about 1220 and 360 before it, and no catch-up afterwards. It looked like lost data in both outage tests.

**Root cause:** the per-pod top list for five minutes before and during the outage showed that whole sources had changed: the window before contained the restart of the consumer (startup logs of `alloy-kafka`, rebalance logs of the broker and the producer); during the outage those pods were simply quiet. The total count measures the cluster's activity, not delivery.

**Fix:** compare per pod, the source against the store, for the same window: `kubectl logs --timestamps --since-time` counted up to the end of the window, against `count_over_time({pod="..."}[...])` in Loki. The three pods checked matched exactly (15/15, 12/12, 1713/1713). The consumer's counters agree: `loki_write_dropped_entries_total` is `0` for every reason.

## `400 timestamp too old` from Loki

**Symptom:** right after Loki came back, the consumer logged:

```
final error sending batch, no retries left, dropping data ... status=400 ...
entry for stream '{container="kube-flannel", ...}' has timestamp too old: 2026-10-14T03:31:57Z,
oldest acceptable timestamp is: 2026-10-18T08:23:22Z
```

**Root cause:** every restart of the producer Alloy makes `loki.source.kubernetes` read the containers' logs from the start, so long-running pods (Flannel, MetalLB, node-exporter) send lines weeks old. Loki rejects anything older than seven days (`reject_old_samples_max_age`).

**Fix:** none needed. These lines were stored when they were first written; only the re-sent copies are rejected. `4xx` responses other than `429` are never retried, so they do not block the queue even with infinite retries.
