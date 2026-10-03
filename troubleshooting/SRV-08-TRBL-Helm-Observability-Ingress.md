---
tags: [homelab, project, kubernetes, helm, troubleshooting]
---

# Troubleshooting — Helm, Observability, and Ingress

> Companion to [Helm, Observability, and Ingress](../server/SRV-08-Helm-Observability-Ingress.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [Helm install script failed to resolve](#helm-install-script-failed-to-resolve) | [Step 1](../server/SRV-08-Helm-Observability-Ingress.md#step-1--helm) |
| [`helm install` interrupted: "cannot re-use a name"](#helm-install-interrupted-cannot-re-use-a-name) | [Step 3](../server/SRV-08-Helm-Observability-Ingress.md#step-3--ingress-controller-ingress-nginx) |

## Helm install script failed to resolve

**Symptom:** the Helm install script failed — `raw.githubusercontent.com` did not resolve.

**Root cause:** the same ISP DNS blocking as in [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md#isp-blocking-public-dns-resolvers).

**Fix:** re-applying the router-DNS fix (netplan `nameservers` → `<GATEWAY_IP>`) restored resolution and the script completed.

## `helm install` interrupted: "cannot re-use a name"

**What happened:** the ingress-nginx install was interrupted with `Ctrl+C` before completion (a follow-up `kubectl get pods` was typed into the same session). Retrying failed:

```
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```

**Root cause:** the release had already been recorded before the interruption and was left in `failed` state, still owning the name `ingress-nginx`.

**Fix:**

```
helm uninstall ingress-nginx -n ingress-nginx
kubectl get pods -n ingress-nginx      # confirmed empty
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

The reinstall was allowed to complete before any further commands were run. See [Guide: Helm Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Helm-Basics.md).

## Related

- [Helm, Observability, and Ingress](../server/SRV-08-Helm-Observability-Ingress.md) — the working procedure
- [SRV-03-TRBL](./SRV-03-TRBL-Network-Configuration.md) — the original ISP DNS-blocking incident
- [Guide: Helm Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Helm-Basics.md) — companion Guides repository
