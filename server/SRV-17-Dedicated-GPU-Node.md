---
tags: [homelab, project, kubernetes, gpu, scheduling, taints, resource-quota]
---

# Dedicated GPU Node

> Status: 🟢 **Node prepared.** The laptop node is labeled and tainted for GPU/AI workloads only, is schedulable again, and has a namespace `ai` with a resource quota waiting for its first workload. No workload runs there yet — see [Open items](#open-items).

Picks up after [Logging and Alerting](./SRV-16-Logging-and-Alerting.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The laptop is `<GPU_HOSTNAME>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Node roles](#node-roles)
4. [Step 1 — Check the node](#step-1--check-the-node)
5. [Step 2 — Label and taint](#step-2--label-and-taint)
6. [Step 3 — Uncordon](#step-3--uncordon)
7. [Step 4 — Namespace, quota and limits](#step-4--namespace-quota-and-limits)
8. [Verification](#verification)
9. [Limits](#limits)
10. [Open items](#open-items)
11. [Files](#files)
12. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Node label `workload=ai` | `<GPU_HOSTNAME>` | What an AI pod's `nodeSelector` matches |
| Node taint `dedicated=ai:NoSchedule` | `<GPU_HOSTNAME>` | Keeps every pod without a matching toleration off the node |
| Namespace `ai` | Cluster | Home for AI workloads; Pod Security `baseline` enforced, `restricted` reported |
| ResourceQuota `ai-quota` | `ai` | Caps what the namespace can request |
| LimitRange `ai-defaults` | `ai` | Default requests and limits for containers that set none |
| ServiceAccount `ai-workload` | `ai` | Identity for AI pods; its API token is not mounted |

Concepts (taint vs selector vs cordon, why order matters): [Guide: Pod Scheduling](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Pod-Scheduling-nodeSelector-Affinity-Cordon.md).

## Why

Until now the laptop was cordoned: no ordinary pod could be placed there, and only its node-level DaemonSet pods ran ([SRV-13](./SRV-13-ArgoCD-GitOps.md#open-items)). That protected the machine, which is also used and administered by another person, but left its GPU unusable for the cluster — the reason the node was added ([SRV-11](./SRV-11-GPU-Laptop-Node-Profile.md)).

A cordon is all-or-nothing. Replacing it with a **taint** keeps the same protection for ordinary pods and lets chosen workloads in.

## Node roles

| Node | Role | How |
|---|---|---|
| Control plane (`<HOSTNAME>`) | Cluster management only | Default taint `node-role.kubernetes.io/control-plane:NoSchedule` |
| Worker (`<WORKER_HOSTNAME>`) | Platform and ordinary applications | The only node that accepts ordinary pods |
| Laptop (`<GPU_HOSTNAME>`) | GPU/AI workloads only | Label `workload=ai` + taint `dedicated=ai:NoSchedule` |

A pod meant for the laptop needs both halves: a toleration for the taint (permission to enter) and `nodeSelector: {workload: ai}` (the pull). The toleration alone would let it run anywhere else too.

## Step 1 — Check the node

```
kubectl get nodes -o wide
kubectl describe node <GPU_HOSTNAME> | grep -A8 -E "Taints|Capacity|Allocatable"
kubectl get pods -A -o wide --field-selector spec.nodeName=<GPU_HOSTNAME>
```

Result before the change: the node was `Ready,SchedulingDisabled` with the single taint `node.kubernetes.io/unschedulable:NoSchedule`; capacity 12 CPUs, about 15 GiB of memory and `nvidia.com/gpu: 1` (the Device Plugin from [SRV-12](./SRV-12-GPU-Node-Setup.md) works). Six pods ran on it, all DaemonSets: Flannel, kube-proxy, the NVIDIA Device Plugin, the two MetalLB pods and node-exporter.

On the laptop itself, `nvidia-smi` showed the RTX 3050 Ti Mobile idle: 12 MiB of 4096 MiB used, the only process being the desktop shell. (`nvidia-smi` has to be run on the laptop; the control plane has no GPU.)

## Step 2 — Label and taint

```
kubectl label node <GPU_HOSTNAME> workload=ai
kubectl taint node <GPU_HOSTNAME> dedicated=ai:NoSchedule
kubectl describe node <GPU_HOSTNAME> | grep -A3 Taints
```

The node now carries two taints: `dedicated=ai:NoSchedule` and the cordon's `node.kubernetes.io/unschedulable:NoSchedule`.

## Step 3 — Uncordon

The taint goes first: uncordoning an untainted node would let any ordinary pod be placed there until the taint arrives.

```
kubectl uncordon <GPU_HOSTNAME>
kubectl describe node <GPU_HOSTNAME> | grep -A3 Taints
```

Result: `dedicated=ai:NoSchedule` only, `Unschedulable: false`.

## Step 4 — Namespace, quota and limits

`~/k8s-manifests/ai-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai
  labels:
    # baseline is enforced; restricted is only reported, so we see what would break
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ai-quota
  namespace: ai
spec:
  hard:
    pods: "5"
    requests.cpu: "6"
    limits.cpu: "6"
    requests.memory: 8Gi
    limits.memory: 8Gi
    requests.nvidia.com/gpu: "1"
    persistentvolumeclaims: "2"
    requests.storage: 40Gi
---
apiVersion: v1
kind: LimitRange
metadata:
  name: ai-defaults
  namespace: ai
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 256Mi
      default:
        cpu: "1"
        memory: 1Gi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-workload
  namespace: ai
automountServiceAccountToken: false
```

| Part | Reason |
|---|---|
| Pod Security `baseline` enforced, `restricted` warn/audit | Many AI images run as root, which `restricted` would block; the warnings show what would break if it is tightened later |
| Quota: 6 CPUs, 8 GiB, 1 GPU | The laptop is also a person's machine: this is the ceiling on what the cluster may take from it |
| LimitRange | A container with no limits gets a modest default, and the quota requires limits to exist |
| No ServiceAccount token | A model server does not need the Kubernetes API |

```
kubectl apply -f ~/k8s-manifests/ai-namespace.yaml
kubectl -n ai get resourcequota,limitrange,serviceaccount
kubectl get ns ai --show-labels
```

`kubectl apply --dry-run=server` accepts the Namespace but answers `NotFound` for the objects inside it, because a dry run does not persist the namespace. This is not a manifest error; an empty namespace with a quota creates no pods, so it was applied directly.

## Verification

| Check | Result |
|---|---|
| `kubectl describe node <GPU_HOSTNAME> \| grep -A3 Taints` | Only `dedicated=ai:NoSchedule`; `Unschedulable: false` |
| `kubectl get nodes --show-labels \| grep <GPU_HOSTNAME>` | Contains `workload=ai` |
| `kubectl get pods -A -o wide --field-selector spec.nodeName=<GPU_HOSTNAME>` | The same six DaemonSet pods; nothing else was placed |
| `kubectl -n ai get resourcequota` | `ai-quota` with the limits above, usage `0` |
| `kubectl get ns ai --show-labels` | The three `pod-security.kubernetes.io/...` labels |

## Limits

- **The quota is a first estimate.** The real load of the laptop owner is in the node-exporter graphs in Grafana, not in `kubectl describe node` (which counts only pod requests).
- **The GPU is whole.** `nvidia.com/gpu: 1` gives the card to one pod entirely; it is not shared unless time-slicing is configured. With 4 GiB of VRAM only small models fit on it.
- **The node is not always on.** The laptop is shut down or sleeps; a pod placed there is unavailable meanwhile, and its volume (`local-path`) lives on that machine's disk.
- **`NetworkPolicy` would not be enforced.** Flannel does not implement it, so network isolation of the `ai` namespace needs a policy-capable CNI (for example Calico) — a separate change.
- **Taints keep strangers out, nothing more.** They do not limit what an admitted pod does; that is what the quota, container limits and Pod Security labels are for.

## Open items

- ⬜ **No workload yet.** The first AI service has to carry `nodeSelector: {workload: ai}`, a toleration for `dedicated=ai` and a `nvidia.com/gpu` limit.
- ⬜ **Remove the laptop filters** in Alloy (the `drop` rule) and Alertmanager (the two `"null"` routes) from [SRV-16](./SRV-16-Logging-and-Alerting.md) when the first workload starts, so its logs and alerts are not hidden.
- ⬜ **Check the real load** of the laptop in Grafana and adjust the quota.
- ⬜ **Network isolation** needs a policy-enforcing CNI.
- ⬜ The laptop stays **out of the Ansible inventory** ([SRV-09](./SRV-09-Ansible-Node-Provisioning.md)) by decision.

## Files

| File | Location | Contents |
|---|---|---|
| `ai-namespace.yaml` | `~/k8s-manifests/` on `<HOSTNAME>` | Namespace `ai`, quota, limit range, ServiceAccount |
| Node label and taint | Node object `<GPU_HOSTNAME>` | `workload=ai`, `dedicated=ai:NoSchedule` (set with `kubectl`, not stored in a file) |

## Related

- [Logging and Alerting](./SRV-16-Logging-and-Alerting.md) — the previous step; the laptop filters live there
- [Node Profile — GPU Laptop Worker](./SRV-11-GPU-Laptop-Node-Profile.md) — what the node is
- [GPU Node Setup](./SRV-12-GPU-Node-Setup.md) — how its GPU reached pods
- [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md) — where the earlier cordon decision was made
- [Guide: Pod Scheduling](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Pod-Scheduling-nodeSelector-Affinity-Cordon.md) — companion Guides repository
- [Guide: Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/kubernetes/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
