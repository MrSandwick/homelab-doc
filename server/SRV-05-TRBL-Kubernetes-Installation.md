---
tags: [homelab, project, kubernetes, troubleshooting]
---

# Troubleshooting — Kubernetes Installation

> Companion to [Kubernetes Installation (kubeadm)](./SRV-05-Kubernetes-Installation.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [`kubeadm init` preflight: `conntrack` not found](#kubeadm-init-preflight-conntrack-not-found) | [Step 5](./SRV-05-Kubernetes-Installation.md#step-5--preflight-dependencies) |
| [Flannel in `CrashLoopBackOff`: `br_netfilter` not loaded](#flannel-in-crashloopbackoff-br_netfilter-not-loaded) | [Step 8](./SRV-05-Kubernetes-Installation.md#step-8--installing-flannel-cni) |
| [Test pod stuck `Pending`: control-plane taint](#test-pod-stuck-pending-control-plane-taint) | [Step 10](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint) |
| [`port-forward` unreachable from the LAN, then "address already in use"](#port-forward-unreachable-from-the-lan-then-address-already-in-use) | [Step 10](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint) |

## `kubeadm init` preflight: `conntrack` not found

**Symptom:** the first `kubeadm init` attempt failed at the preflight stage:

```
[ERROR FileExisting-conntrack]: conntrack not found in system path
```

**Root cause:** `conntrack` is required by `kube-proxy` to track connection state in netfilter and is not installed by default.

**Fix:** installed along with two other commonly-required tools; `kubeadm init` succeeded on the second attempt.

```
sudo apt update
sudo apt install -y conntrack ethtool socat
```

## Flannel in `CrashLoopBackOff`: `br_netfilter` not loaded

**Symptom:** after `kubectl apply` of the Flannel manifest, the Flannel pod (namespace `kube-flannel`, not `kube-system`) entered `CrashLoopBackOff`. Logs:

```
Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory
```

**Root cause:** the `br_netfilter` kernel module had been loaded manually with `modprobe` in Step 3, which does not persist across reboots unless the module is registered for autoload. The server had rebooted at least once since (during earlier networking troubleshooting), silently dropping the module.

**Fix — reload the module and make it persistent:**

```
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo sysctl --system
cat /proc/sys/net/bridge/bridge-nf-call-iptables   # → 1
```

Then the crashing pod was deleted so its DaemonSet would recreate it:

```
kubectl delete pod <kube-flannel-pod-name> -n kube-flannel
```

It came up `Running` on the next attempt. The autoload registration is now part of [Step 3](./SRV-05-Kubernetes-Installation.md#step-3--kernel-modules-and-sysctl-settings) and was applied up front on the second node.

## Test pod stuck `Pending`: control-plane taint

**Symptom:** the `nginx-test` pod stayed `Pending` indefinitely.

```
kubectl describe pod <pod-name>
```

```
0/1 nodes are available: 1 node(s) had untolerated taint
{node-role.kubernetes.io/control-plane: }
```

**Root cause:** `kubeadm init` applies a taint to the control-plane node (`node-role.kubernetes.io/control-plane:NoSchedule`) so ordinary workloads are not scheduled onto it. With only one node in the cluster, there was nowhere else to schedule the pod. See [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md).

**Fix (temporary, single-node only):** the taint was removed for the duration of the test and restored afterwards — see [Step 10](./SRV-05-Kubernetes-Installation.md#step-10--first-test-workload-and-the-control-plane-taint).

## `port-forward` unreachable from the LAN, then "address already in use"

**Symptom:** plain `kubectl port-forward deployment/nginx-test 8080:80` was unreachable from another machine; a retry failed with "address already in use".

**Root cause:** `port-forward` binds to `localhost` only by default, and an earlier `port-forward` process left running in a background terminal was still holding the port.

**Fix:**

```
sudo lsof -i :8080
kill -9 <PID>
kubectl port-forward --address 0.0.0.0 deployment/nginx-test 8080:80
```

## Related

- [Kubernetes Installation (kubeadm)](./SRV-05-Kubernetes-Installation.md) — the working procedure
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — where the `br_netfilter` fix was applied up front
- [Kubernetes Taints and Tolerations](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Kubernetes-Taints-and-Tolerations.md) — companion Guides repository
