---
tags: [homelab, project, kubernetes, gpu, nvidia, troubleshooting]
---

# Troubleshooting — GPU Node Setup

> Companion to [GPU Node Setup — NVIDIA Container Runtime on the Laptop Worker](./SRV-12-GPU-Node-Setup.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [`nvidia-ctk` wrote an empty config on the first attempt](#nvidia-ctk-wrote-an-empty-config-on-the-first-attempt) | [Step 2](./SRV-12-GPU-Node-Setup.md#step-2--configure-containerd-to-use-the-nvidia-runtime) |
| [Device Plugin DaemonSet crashed on the nodes without a GPU](#device-plugin-daemonset-crashed-on-the-nodes-without-a-gpu) | [Step 3](./SRV-12-GPU-Node-Setup.md#step-3--install-the-nvidia-device-plugin) |
| [Device Plugin crashed on the GPU node: wrong default runtime](#device-plugin-crashed-on-the-gpu-node-wrong-default-runtime) | [Step 2](./SRV-12-GPU-Node-Setup.md#step-2--configure-containerd-to-use-the-nvidia-runtime) |

## `nvidia-ctk` wrote an empty config on the first attempt

**Symptom:** the first run of `nvidia-ctk runtime configure --runtime=containerd` (without an explicit `--config` path) reported success and wrote to `/etc/containerd/conf.d/99-nvidia.toml` — but the file contained only `version = 3`, with no `nvidia` runtime section.

**Root cause:** not fully diagnosed.

**Fix:** re-running the same command with the `--config` path explicitly specified produced the full, correct configuration, including the `[plugins."io.containerd.cri.v1.runtime".containerd.runtimes.nvidia]` block. Explicitly passing `--config` is the invocation used in the main doc, to avoid relying on the tool's default path resolution.

## Device Plugin DaemonSet crashed on the nodes without a GPU

**Symptom:** after applying the static Device Plugin manifest, the pods on the two CPU-only nodes (`<HOSTNAME>`, `<WORKER_HOSTNAME>`) immediately entered `CrashLoopBackOff`.

**Root cause:** the static manifest has no awareness of which nodes have a GPU — it schedules a pod on every node in the cluster.

**Fix — label the GPU node and restrict the DaemonSet to it:**

```bash
kubectl label node <GPU_HOSTNAME> accelerator=nvidia-gpu
kubectl patch daemonset nvidia-device-plugin-daemonset -n kube-system \
  -p '{"spec": {"template": {"spec": {"nodeSelector": {"accelerator": "nvidia-gpu"}}}}}'
```

The pods on the two non-GPU nodes were removed automatically once the `nodeSelector` no longer matched them.

## Device Plugin crashed on the GPU node: wrong default runtime

**Symptom:** even restricted to `<GPU_HOSTNAME>`, the plugin pod kept crashing.

```bash
kubectl logs -n kube-system <nvidia-device-plugin-pod-name>
```

```
E... factory.go:99] Failed to initialize NVML: ERROR_LIBRARY_NOT_FOUND.
E... factory.go:100] If this is a GPU node, did you set the docker default runtime to `nvidia`?
```

**Root cause:** the `nvidia` runtime existed in containerd's configuration, but containerd was still launching containers with the default `runc` runtime, which has no access to the NVIDIA driver libraries — defining a runtime does not make it the default.

**Fix — on the GPU node**, set `nvidia` as the default runtime:

```bash
sudo nano /etc/containerd/conf.d/99-nvidia.toml
```

```toml
[plugins."io.containerd.cri.v1.runtime".containerd]
  default_runtime_name = "nvidia"
```

```bash
sudo systemctl restart containerd
```

Then, from the control plane, the crashing pod was deleted so the DaemonSet would recreate it under the new config:

```bash
kubectl delete pod -n kube-system <nvidia-device-plugin-pod-name>
```

It came up `Running` with zero restarts on the next attempt.

## Related

- [GPU Node Setup](./SRV-12-GPU-Node-Setup.md) — the working procedure
- [Node Profile — GPU Laptop Worker](./SRV-11-GPU-Laptop-Node-Profile.md)
