---
tags: [homelab, project, kubernetes, gateway-api, envoy-gateway, metallb, cert-manager, letsencrypt, tls, dns, cloudflare]
---

# Public Gateway and HTTPS

> Status: 🟢 **Working.** A second Gateway, `public`, has its own MetalLB address and carries internet traffic only; cert-manager issues Let's Encrypt certificates for `<DOMAIN>` and `www.<DOMAIN>` over HTTP-01, and the Gateway terminates HTTPS with them. Staging issuance was tested first, then production. 🟡 Limit: DNS is hosted at Cloudflare in **DNS-only** mode, so Cloudflare is a name-server dependency but not in the traffic path — see [Limits](#limits).

Picks up after [Alerting Reliability](./SRV-22-Alerting-Reliability.md) and, on the network side, [Public Port Forwarding](../network/NET-09-Public-Port-Forwarding.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the admin user `<USER>`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — The public Gateway](#step-1--the-public-gateway)
5. [Step 2 — Reachability test](#step-2--reachability-test)
6. [Step 3 — cert-manager](#step-3--cert-manager)
7. [Step 4 — Let's Encrypt issuers](#step-4--lets-encrypt-issuers)
8. [Step 5 — HTTPS listeners and certificates](#step-5--https-listeners-and-certificates)
9. [Step 6 — DNS](#step-6--dns)
10. [Verification](#verification)
11. [Limits](#limits)
12. [Files](#files)
13. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| `EnvoyProxy` `public-proxy` | `envoy-gateway-system` | Proxy settings for the public Gateway: `externalTrafficPolicy: Local` |
| `Gateway` `public` | `envoy-gateway-system`, address `<PUBLIC_GW_IP>` from the MetalLB pool | Listeners: `http` (80), `https-apex` and `https-www` (443); accepts routes only from namespaces labelled `homelab/public-gateway: allowed` |
| cert-manager (Helm release `cert-manager`, chart v1.21.3) | `cert-manager` | Requests and renews certificates; `config.enableGatewayAPI` lets it solve challenges through an `HTTPRoute` and read Gateways |
| `ClusterIssuer` `letsencrypt-staging`, `letsencrypt-prod` | cluster-wide | Let's Encrypt accounts with an HTTP-01 solver that attaches a temporary route to the `public` Gateway |
| `Certificate` `<DOMAIN_SLUG>-tls`, `www-<DOMAIN_SLUG>-tls` | `envoy-gateway-system` | Created by cert-manager from the Gateway annotation; stored in Secrets of the same names |
| DNS records | Cloudflare, DNS-only | `A <DOMAIN>` → public IP; `CNAME www` → `<DOMAIN>` |

## Why

The portfolio had to leave Firebase without using a tunnel service. That needs a public address, a certificate and a Gateway that is separate from the LAN one. The LAN `homelab` Gateway serves Grafana on `/` and ArgoCD under `/argocd` and must never be reachable from the internet.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Tunnel (Cloudflare Tunnel) | Rejected | A third-party service in the traffic path; the whole migration was to remove such dependencies |
| Reuse the LAN Gateway | No — a second Gateway on its own address | Port forwards can then never reach Grafana or ArgoCD, whatever routes are added later |
| Who may attach routes | A namespace label (`homelab/public-gateway: allowed`) | A new application is public only if its namespace is explicitly labelled |
| `externalTrafficPolicy` | `Local` | Keeps the visitor's source address (no SNAT by kube-proxy) for the rate limits and the API's own limits; MetalLB in L2 mode then announces the address only from a node that runs a ready Envoy pod |
| Certificate challenge | HTTP-01 | Needs only port 80, which the test in [NET-09](../network/NET-09-Public-Port-Forwarding.md#step-5--test-from-outside) proved open; DNS-01 would have needed a DNS API token in the cluster just for issuing |
| First issuer | Staging, then production | Staging has generous limits and untrusted certificates; a wrong setup cannot burn the production rate limit |
| How certificates are created | Gateway annotation `cert-manager.io/cluster-issuer` (gateway-shim) | One certificate per HTTPS listener, no separate `Certificate` objects to keep in sync |
| Cloudflare proxy (orange cloud) | Not used; **DNS only** | The proxy would decrypt all traffic at a third party. Using it would also need SSL mode Full (strict), a `ClientTrafficPolicy` to recover visitor IPs and a restriction of the Gateway to Cloudflare's address ranges; the home address is also already public in DNS history. Decided against |
| Public ACME email | Placeholder in Git | The `homelab-gitops` repository is public; the real address is only in the cluster |

## Step 1 — The public Gateway

An unused address from the MetalLB pool (`kubectl -n metallb-system get ipaddresspool -o yaml` and `kubectl get svc -A | grep LoadBalancer`) becomes `<PUBLIC_GW_IP>`:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: public-proxy
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        externalTrafficPolicy: Local
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy
  infrastructure:
    annotations:
      metallb.universe.tf/loadBalancerIPs: <PUBLIC_GW_IP>
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: public-proxy
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              homelab/public-gateway: "allowed"
```

```
kubectl apply -f public-gateway.yaml
kubectl wait --timeout=3m -n envoy-gateway-system gateway/public --for=condition=Programmed
kubectl get gateway -n envoy-gateway-system
```

`public` shows `<PUBLIC_GW_IP>` and `PROGRAMMED True`; `homelab` is unchanged.

## Step 2 — Reachability test

For the test only, a second listener on 443 spoke plain HTTP, and a throwaway namespace (labelled, Pod Security `restricted`) ran one nginx pod behind an `HTTPRoute`. It answered on both ports inside the LAN, then the port forwards of [NET-09](../network/NET-09-Public-Port-Forwarding.md) were added and the test repeated from a phone on cellular data. The namespace was deleted afterwards (`kubectl delete namespace reach-test`).

## Step 3 — cert-manager

`platform/cert-manager/values.yaml` in `homelab-gitops`:

```yaml
crds:
  enabled: true
  keep: true
config:
  apiVersion: controller.config.cert-manager.io/v1alpha1
  kind: ControllerConfiguration
  enableGatewayAPI: true
resources:
  requests: {cpu: 10m, memory: 64Mi}
  limits: {memory: 256Mi}
webhook:
  resources:
    requests: {cpu: 10m, memory: 32Mi}
    limits: {memory: 128Mi}
cainjector:
  resources:
    requests: {cpu: 10m, memory: 64Mi}
    limits: {memory: 256Mi}
```

```
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm search repo jetstack/cert-manager --versions | head -5
helm upgrade --install cert-manager jetstack/cert-manager -n cert-manager --create-namespace \
  --version v1.21.3 -f platform/cert-manager/values.yaml
kubectl -n cert-manager get pods
```

Three pods (controller, webhook, cainjector) reach `Running`. `crds.keep: true` keeps the CRDs, and with them every `Certificate`, if the release is ever uninstalled. The Gateway API CRDs were already installed ([SRV-20](./SRV-20-Gateway-API-Envoy-Gateway.md)), which `enableGatewayAPI` needs at start.

## Step 4 — Let's Encrypt issuers

`platform/cert-manager/clusterissuers.yaml` (`<ACME_EMAIL>` is the address Let's Encrypt sends expiry warnings to; in Git it stays a placeholder):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: <ACME_EMAIL>
    privateKeySecretRef:
      name: letsencrypt-staging-account
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - kind: Gateway
                name: public
                namespace: envoy-gateway-system
---
# letsencrypt-prod: the same with server https://acme-v02.api.letsencrypt.org/directory
# and privateKeySecretRef letsencrypt-prod-account
```

The temporary challenge route is created in the namespace of the Gateway, so that namespace has to be allowed to attach:

```
kubectl label namespace envoy-gateway-system homelab/public-gateway=allowed
kubectl get clusterissuer
```

Both issuers show `READY True`.

## Step 5 — HTTPS listeners and certificates

The Gateway gets the issuer annotation and one HTTPS listener per name:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # letsencrypt-staging during the first test
spec:
  listeners:
    - name: http                 # unchanged
    - name: https-apex
      protocol: HTTPS
      port: 443
      hostname: <DOMAIN>
      tls:
        mode: Terminate
        certificateRefs:
          - name: <DOMAIN_SLUG>-tls
      allowedRoutes: {namespaces: {from: Selector, selector: {matchLabels: {homelab/public-gateway: "allowed"}}}}
    - name: https-www
      protocol: HTTPS
      port: 443
      hostname: www.<DOMAIN>
      tls:
        mode: Terminate
        certificateRefs:
          - name: www-<DOMAIN_SLUG>-tls
      allowedRoutes: {namespaces: {from: Selector, selector: {matchLabels: {homelab/public-gateway: "allowed"}}}}
```

Order used: a test name (`new.<DOMAIN>`) with `letsencrypt-staging` first. After `kubectl -n envoy-gateway-system get certificate` showed `READY True`, the annotation was set to production, the `Certificate` and its Secret deleted, and the production certificate was issued within a minute:

```
kubectl -n envoy-gateway-system annotate gateway public cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite
kubectl -n envoy-gateway-system delete certificate new-<DOMAIN_SLUG>-tls
kubectl -n envoy-gateway-system delete secret new-<DOMAIN_SLUG>-tls --ignore-not-found
```

Then the listener moved to the real names (`https-apex`, `https-www`); both certificates were issued in about three minutes. The `new.` name was removed afterwards.

## Step 6 — DNS

In Cloudflare, both records **DNS only** (grey cloud):

| Type | Name | Content |
|---|---|---|
| `A` | `@` | `<PUBLIC_IP>` |
| `CNAME` | `www` | `<DOMAIN>` |

The old Firebase records were deleted. With the cloud orange, Cloudflare would connect to the origin over HTTPS itself and show error 525 while the origin had no certificate ([SRV-23-TRBL](../troubleshooting/SRV-23-TRBL-Public-Gateway-and-HTTPS.md#error-525-from-cloudflare)). Keeping the `A` record current when the address changes is covered in [SRV-25](./SRV-25-Public-Site-Operations.md).

## Verification

```
getent hosts <DOMAIN> www.<DOMAIN>                      # both → <PUBLIC_IP>
kubectl -n envoy-gateway-system get certificate         # both READY True
curl -sv https://<DOMAIN>/ -o /dev/null 2>&1 | grep -E "issuer|HTTP/"
```

The issuer line shows `O=Let's Encrypt` without `(STAGING)`, and the response is `HTTP/2`. From a phone on cellular data the site opens with a padlock.

## Limits

- **Cloudflare is still the DNS host.** Without the proxy it carries no traffic, but if it were unavailable, the name would stop resolving after caches expire.
- **Port 80 must stay open** for HTTP-01 renewals. If the ISP ever blocks inbound 80, the issuers have to move to DNS-01, which needs a DNS API token in the cluster.
- **Certificates renew automatically.** The production certificates issued here are valid for 90 days and cert-manager renews them with about a third of the lifetime left, roughly 30 days before expiry (`Renewal Time` in `kubectl describe certificate`); the staging certificate of the first test was shorter (64 days). An alert fires when under 14 days remain ([SRV-25](./SRV-25-Public-Site-Operations.md)).
- **The staging and production issuers share one Gateway annotation.** Changing the annotation only affects certificates created afterwards; an existing one has to be deleted to be reissued.
- **Anything in a labelled namespace becomes public.** The label is the only gate.
- **Helm release values are in Git, the release itself was installed by hand**, like the other platform releases ([SRV-18](./SRV-18-Backups-and-Platform-Config.md)).

## Files

| File | Location | Contents |
|---|---|---|
| `public-envoyproxy.yaml`, `public-gateway.yaml` | `platform/envoy-gateway/` in `homelab-gitops` | `EnvoyProxy` and the `public` Gateway with its listeners |
| `values.yaml`, `clusterissuers.yaml` | `platform/cert-manager/` | cert-manager Helm values; the two issuers |
| Secrets `<DOMAIN_SLUG>-tls`, `www-<DOMAIN_SLUG>-tls` | `envoy-gateway-system`, not in Git | Certificates and keys |
| Secrets `letsencrypt-*-account` | `cert-manager`, not in Git | ACME account keys |

## Related

- [Public Port Forwarding](../network/NET-09-Public-Port-Forwarding.md) — the router chain in front of this Gateway
- [Gateway API with Envoy Gateway](./SRV-20-Gateway-API-Envoy-Gateway.md) — the LAN Gateway and the controller
- [Portfolio Hosting](./SRV-24-Portfolio-Hosting.md) — the application served through this Gateway
- [Public Site Operations](./SRV-25-Public-Site-Operations.md) — keeping the address, alerts and limits
- [SRV-23-TRBL](../troubleshooting/SRV-23-TRBL-Public-Gateway-and-HTTPS.md) — troubleshooting for this doc
- [Guide: cert-manager and Let's Encrypt](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-cert-manager-and-Lets-Encrypt.md) — companion Guides repository
- [Guide: Gateway API and Envoy Gateway](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-Gateway-API-and-Envoy-Gateway.md) — companion Guides repository
