---
tags: [homelab, project, kubernetes, argocd, gitops, troubleshooting]
---

# Troubleshooting — ArgoCD and GitOps

> Companion to [ArgoCD and GitOps](../server/SRV-13-ArgoCD-GitOps.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [ArgoCD pods scheduled on an unexpected node](#argocd-pods-scheduled-on-an-unexpected-node) | [Step 2](../server/SRV-13-ArgoCD-GitOps.md#step-2--pod-placement) |
| [GitHub push: password authentication rejected](#github-push-password-authentication-rejected) | [Step 5](../server/SRV-13-ArgoCD-GitOps.md#step-5--push-access) |
| [GitHub push: 403 with a fine-grained token](#github-push-403-with-a-fine-grained-token) | [Step 5](../server/SRV-13-ArgoCD-GitOps.md#step-5--push-access) |

## ArgoCD pods scheduled on an unexpected node

**What happened:** after `helm install`, `kubectl get pods -n argocd -o wide` showed every ArgoCD pod on the laptop node (`<GPU_HOSTNAME>`, pod IPs in `10.244.2.x`), not on the OptiPlex worker.

**Root cause:** the install assumed a two-node cluster in which the control-plane taint leaves the worker as the only schedulable node. The taint excludes only the control plane. The laptop had already joined as a third node, carries no taint, and was a valid scheduling target. At the time it was not listed in these docs or in the Ansible inventory (it is now documented in [Node Profile — GPU Laptop Worker](../server/SRV-11-GPU-Laptop-Node-Profile.md)).

**Fix:**

```
kubectl cordon <GPU_HOSTNAME>
kubectl delete pods --all -n argocd
kubectl get pods -n argocd -o wide
```

`cordon` blocks new scheduling without affecting running pods; the deleted pods were recreated by their controllers on `<WORKER_HOSTNAME>` (`NODE` column).

## GitHub push: password authentication rejected

**Symptom:**

```
remote: Invalid username or token. Password authentication is not supported for Git operations.
```

**Root cause:** GitHub does not accept the account password for git over HTTPS.

**Fix:** authenticate with a Personal Access Token.

## GitHub push: 403 with a fine-grained token

**Symptom:**

```
403 ... Permission to <repo> denied
```

**Root cause:** the fine-grained token authenticated correctly but had no repository permissions (`Repositories 0` in the permissions panel). The **Contents** permission was missed because the permission list is alphabetical and scrolls; it is found through the search box.

**Fix:** Settings → Developer settings → Fine-grained tokens → Edit → **Contents: Read and write**, then push again. Permissions update in place; the same token is reused.

## Related

- [ArgoCD and GitOps](../server/SRV-13-ArgoCD-GitOps.md) — the working procedure
- [Node Profile — GPU Laptop Worker](../server/SRV-11-GPU-Laptop-Node-Profile.md) — the third node the pods first landed on
- [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-GitOps-and-ArgoCD.md) — companion Guides repository
