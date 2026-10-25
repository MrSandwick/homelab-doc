---
tags: [homelab, project, kubernetes, kafka, strimzi, kraft, alloy, loki, logging, prometheus, grafana, gitops]
---

# Kafka with Strimzi and a Log Pipeline

> Status: 🟢 **Working.** A single-node Apache Kafka 4.3.1 cluster (KRaft, no ZooKeeper) runs under the Strimzi operator, with its metrics in Prometheus and four Strimzi dashboards in Grafana. Pod logs now travel **Alloy → Kafka topic `logs` → second Alloy → Loki**; labels and dashboards are unchanged, and a five-minute Loki outage lost no log lines. 🟡 Limits: one broker, and while Loki is down the backlog is held in the consumer's memory — see [Limits](#limits).

Picks up after [Gateway API with Envoy Gateway](./SRV-20-Gateway-API-Envoy-Gateway.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. Everything Kafka-related is scheduled on the worker (`<WORKER_HOSTNAME>`), the only schedulable node. Problems hit along the way: [SRV-21-TRBL](../troubleshooting/SRV-21-TRBL-Kafka-Log-Pipeline.md).

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Operator](#step-1--operator)
5. [Step 2 — Kafka cluster](#step-2--kafka-cluster)
6. [Step 3 — Smoke test](#step-3--smoke-test)
7. [Step 4 — Metrics and dashboards](#step-4--metrics-and-dashboards)
8. [Step 5 — Topic for logs](#step-5--topic-for-logs)
9. [Step 6 — Consumer Alloy](#step-6--consumer-alloy)
10. [Step 7 — Producer Alloy](#step-7--producer-alloy)
11. [Step 8 — Loki outage test](#step-8--loki-outage-test)
12. [Verification](#verification)
13. [Limits](#limits)
14. [Files](#files)
15. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Strimzi operator 1.3.0 (Helm release `strimzi`, chart `strimzi/strimzi-kafka-operator`) | Namespace `kafka` | Turns `Kafka`, `KafkaNodePool`, `KafkaTopic` objects into pods, Services, volumes and topics |
| `Kafka homelab` + `KafkaNodePool dual-role` | `kafka` | One pod `homelab-dual-role-0` acting as both KRaft controller and broker; 10 Gi volume on `local-path` |
| Entity operator | `kafka` | Topic and user operators: topics are created from `KafkaTopic` manifests |
| Kafka Exporter | `kafka` | Consumer-group lag and topic offsets as Prometheus metrics |
| PodMonitors (3) | `kafka` | Tell Prometheus to scrape the broker, the entity operator and the cluster operator |
| Dashboards (4 ConfigMaps) | `kafka` | Strimzi Kafka, KRaft, Kafka Exporter and Operators dashboards, loaded by the Grafana sidecar |
| `KafkaTopic logs` | `kafka` | Buffer for pod logs: 3 partitions, retention 3 days |
| Grafana Alloy (release `alloy`) — **producer** | `monitoring` | Reads pod logs as before, now publishes them to topic `logs` |
| Grafana Alloy (release `alloy-kafka`) — **consumer** | `monitoring` | Reads topic `logs` in consumer group `alloy-consumer`, pushes to Loki |

The bootstrap address inside the cluster is `homelab-kafka-bootstrap.kafka.svc.cluster.local:9092` (plain) and `:9093` (TLS).

## Why

Two reasons, one practical and one for learning.

- **Decoupling the log path.** Before, Alloy pushed straight to Loki: if Loki was down or restarting, collection depended on Alloy's own retry buffer. With a broker in between, the collector only has to reach Kafka; the store can be down, slow or replaced without the collector noticing.
- **Event streaming is a core platform skill.** Kafka is the standard event bus in enterprise systems; Strimzi is the CNCF project for running it on Kubernetes with an operator, the same declarative model as everything else in the cluster.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Deployment | Strimzi operator | Declarative Kafka on Kubernetes (CNCF incubating); topics, users and upgrades as objects in Git instead of scripts |
| Metadata mode | KRaft, no ZooKeeper | Kafka 4.x removed ZooKeeper; Strimzi 1.x supports only KRaft |
| Topology | One node pool with roles `controller` + `broker`, one replica | One schedulable worker. Separate controller and broker pools are the production layout; on one node they would only add pods |
| Replication | Every replication factor and `min.insync.replicas` set to 1 | With one broker, a factor of 2 or 3 would leave internal topics unable to be created |
| Storage | `persistent-claim` 10 Gi on `local-path`, `deleteClaim: false` | Messages survive pod restarts; the volume is kept if the cluster object is deleted |
| Memory | Request and limit 2 Gi, heap 1 Gi | Kafka uses the rest of the container memory as page cache; fits next to the monitoring stack on the worker |
| Listeners | Internal only: `plain` 9092, `tls` 9093 | Nothing outside the cluster talks to Kafka |
| Metrics | Strimzi Metrics Reporter (`metricsConfig.type: strimziMetricsReporter`) + Kafka Exporter | The current Strimzi way, no JMX exporter sidecar; lag comes from the exporter |
| Log pipeline pattern | Grafana's documented "Alloy + Kafka" layout: `otelcol.exporter.kafka` → topic → `otelcol.receiver.kafka` | Taken from the Loki documentation instead of a home-made format; messages are OTLP protobuf |
| Back into Loki | `otelcol.exporter.loki` → `loki.write`, not Loki's OTLP endpoint | Keeps the old label names (`namespace`, `pod`, `container`, `node`), so dashboards and alert queries need no change ([why](../troubleshooting/SRV-21-TRBL-Kafka-Log-Pipeline.md#logs-arrive-under-unknown_service-without-labels)) |
| Consumer as a second Alloy | Separate release `alloy-kafka` | Producer and consumer fail and scale independently; either can be restarted without touching the other |
| Chart version | Both Alloy releases on chart `1.13.1` (Alloy v1.20.1) | One version to read the reference for |

## Step 1 — Operator

```
helm repo add strimzi https://strimzi.io/charts/
helm repo update
helm install strimzi strimzi/strimzi-kafka-operator -n kafka --create-namespace
kubectl wait --timeout=3m -n kafka deployment/strimzi-cluster-operator --for=condition=Available
```

What this operator version supports is read from the cluster rather than from memory:

```
kubectl get crd kafkas.kafka.strimzi.io -o jsonpath='{.spec.versions[*].name}{"\n"}'
kubectl -n kafka get deployment strimzi-cluster-operator -o yaml | grep -A6 STRIMZI_KAFKA_IMAGES
```

API version `v1`; Kafka 4.2.x and 4.3.x. The newest, 4.3.1, is used.

## Step 2 — Kafka cluster

`platform/kafka/kafka.yaml` in `homelab-gitops` (final form, with the metrics settings from Step 4):

```
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
  namespace: kafka
  labels:
    strimzi.io/cluster: homelab
spec:
  replicas: 1
  roles: [controller, broker]
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        class: local-path
        deleteClaim: false
  resources:
    requests: {cpu: 500m, memory: 2Gi}
    limits: {memory: 2Gi}
  jvmOptions:
    -Xms: 1g
    -Xmx: 1g
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: homelab
  namespace: kafka
spec:
  kafka:
    version: 4.3.1
    metricsConfig:
      type: strimziMetricsReporter
    listeners:
      - {name: plain, port: 9092, type: internal, tls: false}
      - {name: tls, port: 9093, type: internal, tls: true}
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator: {}
    userOperator: {}
  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
```

```
kubectl apply -f platform/kafka/kafka.yaml
kubectl wait kafka/homelab -n kafka --for=condition=Ready --timeout=10m
kubectl get pods,pvc -n kafka
```

The `Kafka` object becomes `Ready` after a few minutes (image pulls). Pods: the operator, `homelab-dual-role-0`, the entity operator and the Kafka Exporter; one PVC `Bound` at 10 Gi.

## Step 3 — Smoke test

A topic from a manifest, one message in, the same message out, using the Kafka image that matches the operator:

```
kubectl apply -f - <<'YAML'
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: test
  namespace: kafka
  labels:
    strimzi.io/cluster: homelab
spec:
  partitions: 3
  replicas: 1
YAML

kubectl -n kafka run kafka-producer --rm -i --restart=Never \
  --image=quay.io/strimzi/kafka:1.3.0-kafka-4.3.1 -- \
  bash -c 'echo "hello from kafka" | bin/kafka-console-producer.sh --bootstrap-server homelab-kafka-bootstrap:9092 --topic test'

kubectl -n kafka run kafka-consumer --rm -i --restart=Never \
  --image=quay.io/strimzi/kafka:1.3.0-kafka-4.3.1 -- \
  bin/kafka-console-consumer.sh --bootstrap-server homelab-kafka-bootstrap:9092 --topic test --from-beginning --max-messages 1
```

The consumer prints `hello from kafka` and `Processed a total of 1 messages`.

## Step 4 — Metrics and dashboards

The PodMonitors and dashboards come from the examples shipped with the Strimzi 1.3.0 release (`examples/metrics/`), downloaded on the control plane, not written by hand.

- **PodMonitors** (`platform/kafka/monitoring/`): `kafka-resources-metrics`, `entity-operator-metrics`, `cluster-operator-metrics`. Changes from the example: namespace `kafka`, and the label `release: prometheus` that `kube-prometheus-stack` uses to select monitors.
- **Dashboards** (`platform/kafka/dashboards/`): `strimzi-kafka`, `strimzi-kraft`, `strimzi-kafka-exporter`, `strimzi-operators`, each a ConfigMap with the label `grafana_dashboard: "1"` so the Grafana sidecar loads it. The `${DS_PROMETHEUS}` placeholders in the JSON were replaced with the uid of the existing data source, `prometheus`.

```
kubectl apply -f platform/kafka/monitoring/ -f platform/kafka/dashboards/
kubectl apply -f platform/kafka/kafka.yaml    # metricsConfig + kafkaExporter, rolls the broker
```

In Prometheus (Status → Targets) the three `podMonitor/kafka/...` pools are `up`; the four dashboards appear in Grafana and show data.

## Step 5 — Topic for logs

`platform/kafka/topics/logs.yaml`:

```
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: logs
  namespace: kafka
  labels:
    strimzi.io/cluster: homelab
spec:
  partitions: 3
  replicas: 1
  config:
    retention.ms: 259200000     # 3 days
```

```
kubectl apply -f platform/kafka/topics/logs.yaml
kubectl get kafkatopic -n kafka
```

`READY True` a few seconds after creation. Three partitions let up to three consumers share the topic later; three days is far longer than any outage this pipeline has to bridge.

## Step 6 — Consumer Alloy

The consumer is installed first, so that nothing is published to a topic nobody reads. `platform/alloy-kafka/values.yaml`:

```
controller:
  type: deployment
  replicas: 1

alloy:
  configMap:
    content: |
      otelcol.receiver.kafka "default" {
        brokers          = ["homelab-kafka-bootstrap.kafka.svc.cluster.local:9092"]
        protocol_version = "2.0.0"
        group_id         = "alloy-consumer"
        initial_offset   = "earliest"

        // commit the offset only after the message was forwarded
        message_marking {
          after_execution = true
        }

        logs {
          topics   = ["logs"]
          encoding = "otlp_proto"
        }

        output {
          logs = [otelcol.processor.attributes.labels.input]
        }
      }

      // which log attributes become Loki labels
      otelcol.processor.attributes "labels" {
        action {
          key    = "loki.attribute.labels"
          action = "insert"
          value  = "namespace, pod, container, node, job, instance"
        }
        output {
          logs = [otelcol.processor.batch.default.input]
        }
      }

      otelcol.processor.batch "default" {
        output {
          logs = [otelcol.exporter.loki.default.input]
        }
      }

      otelcol.exporter.loki "default" {
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url                 = "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"
          max_backoff_retries = 0   // retry forever while Loki is down
        }
      }
```

```
helm upgrade --install alloy-kafka grafana/alloy -n monitoring --version 1.13.1 \
  -f platform/alloy-kafka/values.yaml
kubectl -n monitoring rollout status deploy/alloy-kafka
```

Key points:

- In Alloy v1.20 the topic and encoding go in the `logs { }` block; `topic`/`encoding` at the top level are rejected.
- `loki.attribute.labels` is a hint read by `otelcol.exporter.loki`: only the attributes it names become Loki labels. Without it every line lands in one stream `service_name="unknown_service"`.
- `max_backoff_retries = 0` means "retry forever" (the default 10 retries gives up after roughly eight and a half minutes of backoff). Responses `4xx` other than `429` are never retried — for example lines older than Loki's 7-day acceptance window — so they cannot block the queue.

## Step 7 — Producer Alloy

The existing Alloy keeps its discovery and relabel rules (including the `drop` rule for the laptop node). Only the destination changes; `platform/alloy/values.yaml`, end of the config:

```
      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [otelcol.receiver.loki.kafka.receiver]
      }

      // convert Loki entries to OTLP and publish them to Kafka
      otelcol.receiver.loki "kafka" {
        output {
          logs = [otelcol.exporter.kafka.default.input]
        }
      }

      otelcol.exporter.kafka "default" {
        brokers          = ["homelab-kafka-bootstrap.kafka.svc.cluster.local:9092"]
        protocol_version = "2.0.0"

        logs {
          topic    = "logs"
          encoding = "otlp_proto"
        }
      }
```

The switch was made in two passes. First `forward_to` listed both the old `loki.write` and the Kafka branch, to see data arrive in the topic while the old path still worked. Then the direct path was removed and its `loki.write` block deleted.

```
helm upgrade alloy grafana/alloy -n monitoring --version 1.13.1 -f platform/alloy/values.yaml
kubectl -n monitoring rollout status deploy/alloy
```

Checks used at each pass:

```
# messages per partition keep growing
kubectl -n kafka exec homelab-dual-role-0 -- \
  bin/kafka-get-offsets.sh --bootstrap-server localhost:9092 --topic logs

# the consumer keeps up: LAG close to 0
kubectl -n kafka exec homelab-dual-role-0 -- \
  bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group alloy-consumer

# label names in Loki are the old ones
kubectl -n monitoring run curl-tmp --rm -i --restart=Never --quiet --image=curlimages/curl -- \
  curl -s "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/labels?since=5m"
```

Labels: `container`, `instance`, `job`, `namespace`, `node`, `pod`, `service_name`, plus `exporter="OTLP"`, which `otelcol.exporter.loki` adds to every stream.

## Step 8 — Loki outage test

The point of the broker is that the log store can be unavailable. Test: stop Loki for about five minutes while the pipeline runs, start it again, compare.

```
date -u +"Loki OFF at %H:%M:%S UTC"
kubectl -n monitoring scale statefulset loki --replicas=0
# ~5 minutes
date -u +"Loki ON at %H:%M:%S UTC"
kubectl -n monitoring scale statefulset loki --replicas=1
kubectl -n monitoring rollout status statefulset/loki
```

The decisive check compares, per pod, the lines the pod actually wrote during the outage window (from the kubelet) with the lines Loki holds for the same window:

```
START=<OFF time>; END=<ON time>
kubectl -n <ns> logs <pod> --all-containers --timestamps --since-time=${START}Z \
  | awk -v e="$END" '$1 < e' | wc -l
# Loki: sum(count_over_time({pod="<pod>"}[<window>])) evaluated at END
```

| Pod | Lines written | Lines in Loki |
|---|---|---|
| `argocd-application-controller-0` | 15 | 15 |
| Alloy producer | 12 | 12 |
| `homelab-dual-role-0` (broker) | 1713 | 1713 |

No loss. The consumer's own counters agree: `otelcol_receiver_accepted_log_records_total` and `loki_write_sent_entries_total` differ only by the records in flight, and every `loki_write_dropped_entries_total` reason is `0`.

What the test also showed: during the outage the consumer group's `LAG` stayed at `0`. The backlog was not left in Kafka; the consumer kept reading, and `loki.write` held the entries in memory and retried until Loki answered. A per-minute line count of the whole cluster is not a valid check for this test — see [SRV-21-TRBL](../troubleshooting/SRV-21-TRBL-Kafka-Log-Pipeline.md#the-per-minute-count-shows-a-dip-that-is-not-a-loss).

## Verification

```
kubectl get kafka,kafkanodepool,kafkatopic -n kafka
kubectl get pods -n kafka
helm list -n monitoring | grep alloy
kubectl -n kafka exec homelab-dual-role-0 -- \
  bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group alloy-consumer
```

`homelab` is `Ready`, topics `logs` and `test` are `READY True`, both Alloy releases are on chart `1.13.1`, and the group `alloy-consumer` has one consumer on all three partitions with a lag of a few messages. In Grafana, Explore on Loki shows fresh lines for `{namespace="monitoring"}`, and the Strimzi dashboards show the broker and the consumer lag.

## Limits

- **One broker, replication factor 1.** If the broker pod or the worker is down, producers cannot publish and logs are not collected during that time. Kafka protects against the *store* failing, not against the broker itself failing.
- **The backlog lives in the consumer's memory while Loki is down.** Offsets are committed once a record is handed to `loki.write`, which accepts it immediately. If the `alloy-kafka` pod restarts during a Loki outage, the entries it was holding are lost. A delivery guarantee end to end needs a synchronous last hop (`otelcol.exporter.otlphttp` to Loki's OTLP endpoint without a sending queue, `error_backoff` on the receiver) and the labels configured on Loki's side.
- **Producer restart re-reads container logs.** After a restart, `loki.source.kubernetes` sends lines it already sent; lines older than seven days are rejected by Loki with `400` (logged, harmless), newer ones are deduplicated by Loki because labels, timestamp and text are identical.
- **Everything on one worker.** Kafka, both Alloy pods and Loki share `<WORKER_HOSTNAME>` ([SRV-17](./SRV-17-Dedicated-GPU-Node.md#node-roles)); the broker adds 2 Gi of memory there.
- **Topic `test`** from the smoke test exists only in the cluster, not in Git.

## Files

| File | Location | Contents |
|---|---|---|
| `kafka.yaml` | `platform/kafka/` in `homelab-gitops` | `KafkaNodePool dual-role` and `Kafka homelab` with metrics and Kafka Exporter |
| `kafka-resources-metrics.yaml`, `entity-operator-metrics.yaml`, `cluster-operator-metrics.yaml` | `platform/kafka/monitoring/` | PodMonitors |
| `strimzi-kafka.yaml`, `strimzi-kraft.yaml`, `strimzi-kafka-exporter.yaml`, `strimzi-operators.yaml` | `platform/kafka/dashboards/` | Grafana dashboards as ConfigMaps |
| `logs.yaml` | `platform/kafka/topics/` | Topic for the log pipeline |
| `values.yaml` | `platform/alloy/` | Producer: pod logs → topic `logs` |
| `values.yaml` | `platform/alloy-kafka/` | Consumer: topic `logs` → Loki |
| Helm release `strimzi` | Namespace `kafka` | Installed by hand from `strimzi/strimzi-kafka-operator` (Strimzi 1.3.0) |
| Helm release `alloy-kafka` | Namespace `monitoring` | `grafana/alloy` chart `1.13.1` |

## Related

- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — Loki and the first Alloy, which this changes from a direct push to a Kafka producer
- [Backups and Platform Config](./SRV-18-Backups-and-Platform-Config.md) — `platform/` in `homelab-gitops`
- [Persistent Storage](./SRV-15-Persistent-Storage.md) — `local-path`, which holds the broker's volume
- [Guide: Kafka and Strimzi](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Kafka-and-Strimzi.md) — companion Guides repository
- Grafana documentation: [Loki — Alloy Kafka logs example](https://grafana.com/docs/loki/latest/send-data/alloy/examples/alloy-kafka-logs/), [otelcol.receiver.kafka](https://grafana.com/docs/alloy/v1.20/reference/components/otelcol/otelcol.receiver.kafka.md), [otelcol.exporter.loki](https://grafana.com/docs/alloy/v1.19/reference/components/otelcol/otelcol.exporter.loki.md), [loki.write](https://grafana.com/docs/alloy/v1.20/reference/components/loki/loki.write.md)
