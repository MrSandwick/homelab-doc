---
tags: [homelab, project, kubernetes, gateway-api, cert-manager, letsencrypt, cloudflare, troubleshooting]
---

# Troubleshooting — Public Gateway and HTTPS

> Companion to [Public Gateway and HTTPS](../server/SRV-23-Public-Gateway-and-HTTPS.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [MetalLB pool name assumed, not read](#metallb-pool-name-assumed-not-read) | [Step 1](../server/SRV-23-Public-Gateway-and-HTTPS.md#step-1--the-public-gateway) |
| [No Certificate after the Gateway change](#no-certificate-after-the-gateway-change) | [Step 5](../server/SRV-23-Public-Gateway-and-HTTPS.md#step-5--https-listeners-and-certificates) |
| [Error 525 from Cloudflare](#error-525-from-cloudflare) | [Step 6](../server/SRV-23-Public-Gateway-and-HTTPS.md#step-6--dns) |
| [The ACME email would have gone into a public repository](#the-acme-email-would-have-gone-into-a-public-repository) | [Step 4](../server/SRV-23-Public-Gateway-and-HTTPS.md#step-4--lets-encrypt-issuers) |

## MetalLB pool name assumed, not read

**Symptom:** `kubectl -n metallb-system get ipaddresspool default-pool` answered `ipaddresspools.metallb.io "default-pool" not found`.

**Root cause:** the command assumed the pool was called `default-pool`; it is not.

**Fix:** list the real pool instead of guessing: `kubectl -n metallb-system get ipaddresspool -o yaml | grep -A3 "addresses:"`, which gave the range, and `kubectl get svc -A | grep LoadBalancer` for the addresses already taken. One free address became `<PUBLIC_GW_IP>`.

## No Certificate after the Gateway change

**Symptom:** after the Gateway was meant to get its HTTPS listener and the issuer annotation, `kubectl -n envoy-gateway-system describe certificate new-<DOMAIN_SLUG>-tls` answered `NotFound`.

**What was found:** nothing was wrong with cert-manager (both issuers were `READY True` and had registered with Let's Encrypt). `kubectl describe gateway public` still showed only the old listener, `Observed Generation: 1`, and no Certificate existed. The change had never reached the cluster: a long multi-line command pasted into the terminal was not executed as written.

**Fix:** the manifest was written to a file first and applied from the file:

```
cat > ~/public-gateway-https.yaml <<'EOF'
...
EOF
kubectl apply -f ~/public-gateway-https.yaml
kubectl -n envoy-gateway-system get certificate
```

The Certificate appeared three seconds later, then became `READY True` within a minute.

**Check added to the procedure:** after changing a Gateway, compare `Observed Generation` and the listener names with the file before blaming the controller; a missing Certificate means the annotation never arrived.

## Error 525 from Cloudflare

**Symptom:** `http://new.<DOMAIN>/` showed Cloudflare's **525** page, and kept showing it after the record was changed.

**Root cause:** the new record was proxied (**orange cloud**). Cloudflare then tried HTTPS to the origin, where no certificate existed yet, and failed the handshake. After the first switch to grey the error was still shown, so the record state had to be checked from the server and not from the phone.

**Fix:** the record was switched to **DNS only**. To tell a stale cache from a wrong record, the name was resolved on the server: `getent hosts new.<DOMAIN>` answers the public IP when the record is grey, and an address in Cloudflare's ranges when it is orange. Once the name resolved to the public IP, the Gateway's own 404 page was served.

**Related decision:** the proxy was considered and left off; see the design decisions in [SRV-23](../server/SRV-23-Public-Gateway-and-HTTPS.md#design-decisions).

## The ACME email would have gone into a public repository

**Symptom:** none yet: caught in review of the commit.

**Root cause:** `clusterissuers.yaml` held the real address, and `homelab-gitops` is public.

**Fix:** the commit was prepared with `git cherry-pick -n`, the address replaced by `<ACME_EMAIL>` with `sed`, and `git diff --cached | grep -q <part of the address>` run as a stop check before committing. The real address exists only in the cluster's `ClusterIssuer` objects. The local branch that held the original commit was never pushed.

**Check added to the procedure:** grep the staged diff for personal data before pushing to a public repository.

## Related

- [Public Gateway and HTTPS](../server/SRV-23-Public-Gateway-and-HTTPS.md) — the working procedure
- [NET-09-TRBL](./NET-09-TRBL-Public-Port-Forwarding.md) — the router chain in front of this Gateway
- [Guide: cert-manager and Let's Encrypt](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-cert-manager-and-Lets-Encrypt.md) — companion Guides repository
