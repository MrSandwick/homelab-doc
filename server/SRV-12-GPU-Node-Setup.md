---
tags: [homelab, project, kubernetes, gpu, nvidia]
---

# GPU Node Setup — NVIDIA Container Runtime on the Laptop Worker

> Status: 🟢 Verified end-to-end. A pod scheduled on the laptop worker node can see and use the physical GPU.

This continues from [Node Profile — GPU Laptop Worker](./SRV-11-GPU-Laptop-Node-Profile.md), which covers what `<GPU_HOSTNAME>` is and the trade-offs of running it, and from [Second Node Setup](./SRV-06-Second-Node-Setup.md), whose join process this node followed. This doc covers the additional steps needed to make its GPU actually usable by pods — joining a node is not sufficient by itself; Kubernetes has no awareness of a GPU on a node unless a specific chain of components is set up.

Problems hit during this work are recorded separately in [SRV-12-TRBL](./SRV-12-TRBL-GPU-Node-Setup.md).

## Why this node specifically

Unlike the two headless Server-OS nodes ([Hardware Selection](./SRV-01-Hardware-Selection.md)), this is a Desktop Ubuntu installation on a laptop that doubles as a daily driver — see [Node Profile](./SRV-11-GPU-Laptop-Node-Profile.md) for the swap/memory trade-off discussion specific to running a GUI alongside cluster workloads. The GPU is the reason this machine was added to the cluster at all — see the "Target workloads" list in [Project Overview](./SRV-00-Project-Overview.md) (the self-hosted RAG stack, previously deprioritized, is the direct motivation).

## Step 0 — confirm the NVIDIA driver is already working

```bash
nvidia-smi
```

On this machine, the proprietary driver (`nvidia-driver-595-open`) was already installed and loaded — Ubuntu Desktop installs typically handle this automatically for display purposes, unlike a headless server where it would need to be installed deliberately. Confirmed working:

```
NVIDIA-SMI 595.91.07   Driver Version: 595.91.07   CUDA Version: 13.2
GPU: NVIDIA GeForce RTX 3050 Ti Mobile
```

If this step doesn't show a working driver on a different machine, `sudo ubuntu-drivers install` (not `autoinstall` — that subcommand doesn't exist in this `ubuntu-drivers` version) followed by a reboot is the standard fix.

## Step 1 — install the NVIDIA Container Toolkit

This is the bridge between the host's NVIDIA driver and containers — without it, a container has no way to access the GPU device even if the driver is perfectly installed on the host:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update
sudo apt install -y nvidia-container-toolkit
```

## Step 2 — configure containerd to use the NVIDIA runtime

```bash
sudo nvidia-ctk runtime configure --runtime=containerd --config=/etc/containerd/conf.d/99-nvidia.toml
sudo systemctl restart containerd
```

The `--config` path is passed explicitly: without it, the first run reported success but wrote an empty config — see [SRV-12-TRBL](./SRV-12-TRBL-GPU-Node-Setup.md#nvidia-ctk-wrote-an-empty-config-on-the-first-attempt).

Confirmed correct config:

```bash
cat /etc/containerd/conf.d/99-nvidia.toml
```

```toml
[plugins."io.containerd.cri.v1.runtime".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
    SystemdCgroup = true
```

`SystemdCgroup = true` carried over correctly from the original containerd setup (see [Kubernetes Installation, Step 2](./SRV-05-Kubernetes-Installation.md#step-2--install-and-configure-containerd)) — this matters, since a mismatched cgroup driver between containerd and kubelet is what originally caused node registration failures on the first two nodes.

Defining the `nvidia` runtime does not make containerd use it; it is also set as the **default** runtime in the same file:

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

Without this the Device Plugin cannot load the NVIDIA libraries — see [SRV-12-TRBL](./SRV-12-TRBL-GPU-Node-Setup.md#device-plugin-crashed-on-the-gpu-node-wrong-default-runtime).

## Step 3 — install the NVIDIA Device Plugin

Run from the control-plane node (`<HOSTNAME>`), not the GPU node — this creates a cluster-wide `DaemonSet`:

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/main/deployments/static/nvidia-device-plugin.yml
```

The static manifest schedules a pod on **every** node. The GPU node is labelled and the DaemonSet restricted to it:

```bash
kubectl label node <GPU_HOSTNAME> accelerator=nvidia-gpu
kubectl patch daemonset nvidia-device-plugin-daemonset -n kube-system \
  -p '{"spec": {"template": {"spec": {"nodeSelector": {"accelerator": "nvidia-gpu"}}}}}'
```

Before the restriction, the pods on the two CPU-only nodes crash-looped — see [SRV-12-TRBL](./SRV-12-TRBL-GPU-Node-Setup.md#device-plugin-daemonset-crashed-on-the-nodes-without-a-gpu).

## Step 4 — verification

From the control-plane node:

```bash
kubectl get pods -n kube-system -o wide | grep nvidia
```

```
nvidia-device-plugin-daemonset-xxxxx   1/1   Running   0   ...   <GPU_HOSTNAME>
```

Confirmed the GPU resource registered with the cluster:

```bash
kubectl describe node <GPU_HOSTNAME> | grep -A 10 "Capacity"
```

```
nvidia.com/gpu:  1
```

**End-to-end test — a pod actually using the GPU:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  nodeSelector:
    kubernetes.io/hostname: <GPU_HOSTNAME>
  containers:
  - name: cuda-test
    image: nvidia/cuda:12.4.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
kubectl logs gpu-test
```

The output matched the host's own `nvidia-smi` exactly — same driver version, same GPU model — confirming the full chain (driver → nvidia-container-toolkit → containerd runtime → Device Plugin → pod) works correctly.

**Note on the pod's `CrashLoopBackOff` after a successful run:** this pod's `command` (`nvidia-smi`) runs once and exits — it has no long-running process. Kubernetes treats that exit as a failure to restart-loop on by default, which is expected and not a sign of a real problem for a single-shot diagnostic pod like this one:

```bash
kubectl describe pod gpu-test | grep -A 5 "Last State"
# Exit Code: 0 confirms a clean, successful run
```

Cleaned up:

```bash
kubectl delete pod gpu-test
```

## What this enables going forward

Any future pod that needs GPU access on this cluster requests it the standard Kubernetes way:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

combined with `nodeSelector: { kubernetes.io/hostname: <GPU_HOSTNAME> }` (or a toleration-based scheme, if this node is ever tainted to reserve it for GPU workloads specifically) to ensure it lands on the one node with a GPU.

## Related

- [SRV-12-TRBL](./SRV-12-TRBL-GPU-Node-Setup.md) — troubleshooting for this doc
- [Node Profile — GPU Laptop Worker](./SRV-11-GPU-Laptop-Node-Profile.md) — what this machine is, and the swap/GUI trade-off discussion
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the join process this node followed
- [Project Overview](./SRV-00-Project-Overview.md) — the RAG-stack target workload this GPU access enables
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the `SystemdCgroup` setting this build carried forward correctly
