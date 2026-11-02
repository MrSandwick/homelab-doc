---
tags: [homelab, project, kubernetes, mongodb, nginx, argocd, github-actions, ghcr, gitops, portfolio, pod-security]
---

# Portfolio Hosting

> Status: 🟢 **Working.** The portfolio (React/Vite site, Express API, MongoDB) runs in the `portfolio` namespace and is served at `https://<DOMAIN>` through the public Gateway. Images are built by GitHub Actions, published to GHCR and rolled out by ArgoCD through an automatic tag commit; projects and skills were loaded into the new database; the contact form stores every message before sending the email. 🟡 Limits: one replica of everything, no network policy, and data older than the October 2025 exports could not be recovered — see [Limits](#limits).

Picks up after [Public Gateway and HTTPS](./SRV-23-Public-Gateway-and-HTTPS.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted. The worker is `<WORKER_HOSTNAME>`, the admin user `<USER>`. The application repository is `MrSandwick/Render-Resume`, the GitOps repository `homelab-gitops`.

## Contents

1. [Components](#components)
2. [Why](#why)
3. [Design decisions](#design-decisions)
4. [Step 1 — Images and the build workflow](#step-1--images-and-the-build-workflow)
5. [Step 2 — Namespace and Secrets](#step-2--namespace-and-secrets)
6. [Step 3 — Manifests and the ArgoCD Application](#step-3--manifests-and-the-argocd-application)
7. [Step 4 — Deploy key for the tag update](#step-4--deploy-key-for-the-tag-update)
8. [Step 5 — Load the data](#step-5--load-the-data)
9. [Verification](#verification)
10. [Limits](#limits)
11. [Files](#files)
12. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Namespace `portfolio` | Pod Security `restricted`, label `homelab/public-gateway: allowed` | Everything of the site; the label lets its routes attach to the public Gateway |
| StatefulSet `mongo` | `mongo:7`, one replica, PVC `data-mongo-0` (5 Gi, `local-path`, on the worker) | The database `portfolio` (`projects`, `skills`, `messages`); authentication on |
| Deployment `api` | `ghcr.io/mrsandwick/render-resume-api:<commit>`, port 5050 | Express API (`/api/projects`, `/api/skills`, `/api/contact`, `/health`) |
| Deployment `web` | `ghcr.io/mrsandwick/render-resume-web:<commit>`, nginx-unprivileged on 8080 | Serves the built SPA and proxies `/api` to the `api` Service |
| `HTTPRoute` `portfolio`, `portfolio-www-redirect`, `portfolio-https-redirect` | `portfolio` | `<DOMAIN>` → `web`; `www` → `<DOMAIN>` (301); HTTP → HTTPS (301) |
| Secrets `mongo-root`, `mongo-app`, `portfolio-api`, `portfolio-smtp` | `portfolio`, not in Git | Database accounts, connection string and admin key, SMTP settings |
| Application `portfolio` | `argocd` | Syncs `apps/portfolio` from `homelab-gitops` into the namespace |

```
visitor → public Gateway (TLS) → web (nginx) ─┬─ static files
                                              └─ /api → api (Express) → mongo
```

## Why

The site ran on Firebase Hosting and Functions with the data in Atlas. Moving it into the cluster removes those services, uses the GitOps and CI/CD path already built for `my-site` ([SRV-13](./SRV-13-ArgoCD-GitOps.md), [SRV-14](./SRV-14-CI-CD-GHCR.md)), and makes the public Gateway of [SRV-23](./SRV-23-Public-Gateway-and-HTTPS.md) carry something real.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Database | MongoDB 7 in the cluster | The reason for leaving was to avoid hosted services; the data is a few kilobytes |
| API address in the browser | Same origin: nginx proxies `/api`, `VITE_API_URL` stays empty | No CORS, no build-time URL, one name to certify |
| Images | Two, `web` and `api`, both public on GHCR, tagged with the commit | Same pattern as `my-site`; the cluster pulls without credentials |
| Tag update | The workflow commits the new tags into `apps/portfolio/web.yaml` and `api.yaml` over a deploy key | ArgoCD reads Git; no manual hash |
| Pod hardening | Pod Security `restricted`; non-root; API and web with read-only root filesystem and all capabilities dropped; Mongo keeps a writable root filesystem | The Mongo image writes outside its data directory; the two stateless images do not |
| Mongo accounts | A root account for bootstrapping and a separate app account with `readWrite` on `portfolio` only | The API can never touch other databases |
| Mongo memory | `--wiredTigerCacheSizeGB=0.25`, memory limit 768 Mi | One node shared with monitoring and Kafka |
| Contact messages | Saved to the `messages` collection first, then emailed; an SMTP failure still answers success if the message is stored | A mail outage cannot lose a message |
| `/health` | Answers before the database gate | A liveness probe that depended on Mongo would restart the API during a database outage |
| Trusted proxies | `TRUST_PROXY=3`, sized for Cloudflare → Envoy → nginx → API | Keeps the API's per-visitor rate limits on the real address. The Cloudflare proxy is not used ([SRV-23](./SRV-23-Public-Gateway-and-HTTPS.md#design-decisions)); with only Envoy and nginx in front, the extra hop is harmless, while too few hops would put every visitor behind one address |
| Admin writes | None exposed | The old design needed none either; content changes by re-importing the JSON exports |
| Network policy | None | Flannel does not enforce them ([SRV-05](./SRV-05-Kubernetes-Installation.md)) |

## Step 1 — Images and the build workflow

In the application repository (branch `feat/self-host`, merged by pull request):

- `server/Dockerfile`: `node:22-alpine`, `npm ci --omit=dev`, `USER 1000:1000`.
- `client/Dockerfile`: a build stage (`npm run build`, with `VITE_FEATURED` as a build argument) and a final `nginxinc/nginx-unprivileged` stage with `client/nginx.conf`: SPA fallback, `index.html` never cached, hashed assets cached for a year, `/api/` proxied to `http://api:5050`, `/healthz` for the probes.
- `.github/workflows/build-and-deploy.yml`: a matrix job builds `web` and `api` and pushes `ghcr.io/mrsandwick/render-resume-<name>:<sha>`; a second job checks out `homelab-gitops` over SSH, rewrites the two image lines with `sed`, fails if the tag is not there (`grep -q`), commits and pushes. Same shape as the `my-site` workflow of [SRV-14](./SRV-14-CI-CD-GHCR.md#step-8--automatic-tag-update).
- Code changes: the unused Firebase packages were removed from the API, `/health` moved in front of the database gate, `TRUST_PROXY` is read from the environment, the contact route saves a `Message` before sending, and visitor input is escaped in the email.

Both images were built and run locally with `--read-only --cap-drop ALL`; `/health`, the SPA fallback, the cache headers and the `/api` proxy answered as expected (the proxy with a 502, since no API sat behind it).

After the first build, both GHCR packages were made **public** (profile → Packages → package settings → visibility).

## Step 2 — Namespace and Secrets

```
kubectl create namespace portfolio
kubectl label namespace portfolio homelab/public-gateway=allowed pod-security.kubernetes.io/enforce=restricted

APP_PASS=$(openssl rand -hex 24)
kubectl -n portfolio create secret generic mongo-root --from-literal=username=root --from-literal=password="$(openssl rand -hex 24)"
kubectl -n portfolio create secret generic mongo-app  --from-literal=username=portfolio --from-literal=password="$APP_PASS"
kubectl -n portfolio create secret generic portfolio-api \
  --from-literal=MONGO_URI="mongodb://portfolio:${APP_PASS}@mongo:27017/portfolio?authSource=portfolio" \
  --from-literal=ADMIN_API_KEY="$(openssl rand -hex 32)"
unset APP_PASS
```

The SMTP settings come from the old `server/.env`. Only the six lines are copied to the node (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM`, `MAIL_TO`), the quotes around `MAIL_FROM` are stripped, because `--from-env-file` would keep them, and the temporary file is shredded:

```
sed -i -E "s/^([A-Z_]+)=[\"'](.*)[\"']$/\1=\2/" /tmp/smtp.env
kubectl -n portfolio create secret generic portfolio-smtp --from-env-file=/tmp/smtp.env
shred -u /tmp/smtp.env
```

Secrets are never committed; `homelab-gitops` is public. They are part of the etcd snapshot in the nightly backup.

## Step 3 — Manifests and the ArgoCD Application

`apps/portfolio/` in `homelab-gitops`: `namespace.yaml`, `mongo.yaml` (ConfigMap with an init script that creates the app user, headless Service, StatefulSet with a `volumeClaimTemplate`), `api.yaml`, `web.yaml`, `routes.yaml`; the later files are described in [SRV-25](./SRV-25-Public-Site-Operations.md). The images start with the tag `bootstrap`, which does not exist: the first build replaces it.

The Application is applied once by hand, like `my-site`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: portfolio
  namespace: argocd
  annotations:
    argocd.argoproj.io/compare-options: ServerSideDiff=true
spec:
  project: default
  source:
    repoURL: https://github.com/MrSandwick/homelab-gitops.git
    targetRevision: main
    path: apps/portfolio
  destination:
    server: https://kubernetes.default.svc
    namespace: portfolio
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

The `ServerSideDiff` annotation stops ArgoCD from showing `OutOfSync` for fields the API server fills in by default ([SRV-24-TRBL](../troubleshooting/SRV-24-TRBL-Portfolio-Hosting.md#argocd-shows-outofsync-after-a-successful-sync)).

## Step 4 — Deploy key for the tag update

As in [SRV-14 Step 8](./SRV-14-CI-CD-GHCR.md#step-8--automatic-tag-update): a key pair is generated on the node, the **public** half is added to `homelab-gitops` → Settings → Deploy keys with write access (title `github-actions (render-resume)`), the **private** half is stored as the Actions secret `GITOPS_DEPLOY_KEY` in the application repository, and both files are shredded on the node. The private key is never pasted anywhere else.

## Step 5 — Load the data

The Atlas cluster no longer existed (its address did not resolve), and the local development Mongo volume was empty. The data came from the JSON exports kept in the application repository (`server/db/`: 55 skills, 9 projects, from October 2025). They are copied into the Mongo pod and imported with the app account:

```
kubectl -n portfolio cp /tmp/skills.json mongo-0:/tmp/skills.json
kubectl -n portfolio cp /tmp/projects.json mongo-0:/tmp/projects.json
kubectl -n portfolio exec mongo-0 -- sh -c 'mongoimport --db portfolio --collection skills --jsonArray --file /tmp/skills.json -u "$APP_USER" -p "$APP_PASS" --authenticationDatabase portfolio'
kubectl -n portfolio exec mongo-0 -- sh -c 'mongoimport --db portfolio --collection projects --jsonArray --file /tmp/projects.json -u "$APP_USER" -p "$APP_PASS" --authenticationDatabase portfolio'
kubectl -n portfolio exec mongo-0 -- rm -f /tmp/skills.json /tmp/projects.json
```

`55 document(s) imported successfully` and `9 document(s) imported successfully`. The database name in Atlas was `R&R`; the new one is `portfolio`, and the collections are created by the import.

## Verification

```
kubectl -n portfolio get pods                      # mongo-0, api-…, web-… all 1/1 Running
kubectl -n argocd get application portfolio        # Synced, Healthy
curl -s https://<DOMAIN>/api/health                 # {"ok":true,"env":"production"}
curl -s https://<DOMAIN>/api/projects | head -c 200 # a JSON list
```

`https://<DOMAIN>/` shows the site with projects and skills; `https://www.<DOMAIN>/` and `http://<DOMAIN>/` redirect to it.

## Limits

- **One replica of everything.** A node outage or a pod restart means a short outage; `web` and `api` start in seconds, Mongo in about a minute.
- **Mongo's data is on one disk.** The `local-path` volume lives on the worker; backups are in [SRV-25](./SRV-25-Public-Site-Operations.md).
- **No network policy.** Any pod in the cluster can reach Mongo and the API; the Mongo accounts are the only barrier.
- **The data is the October 2025 export.** Changes made in Atlas after that date could not be recovered.
- **Content changes mean a re-import.** There is no admin interface; projects and skills are edited as JSON and imported.
- **The images are public.** They contain only the built site and the API code, no secrets, but anyone can pull them.
- **The first `bootstrap` tag is a trap.** Copying an old manifest over a CI-written one resets the tag ([SRV-24-TRBL](../troubleshooting/SRV-24-TRBL-Portfolio-Hosting.md#image-tag-reset-to-bootstrap)).

## Files

| File | Location | Contents |
|---|---|---|
| `namespace.yaml`, `mongo.yaml`, `api.yaml`, `web.yaml`, `routes.yaml` | `apps/portfolio/` in `homelab-gitops` | The workload and its routes |
| `Dockerfile`, `nginx.conf` | `client/` in the application repository | Web image |
| `Dockerfile` | `server/` | API image |
| `build-and-deploy.yml` | `.github/workflows/` | Build, push, tag commit |
| `server/db/*.json` | application repository | Exports of skills and projects |
| `application-portfolio.yaml` | applied by hand; kept on the control plane (`~/homelab-staging/` at the time, to be moved next to the `my-site` Application in `~/k8s-manifests/`) | The ArgoCD Application |
| Secrets `mongo-root`, `mongo-app`, `portfolio-api`, `portfolio-smtp` | `portfolio` namespace, not in Git | Credentials |

## Related

- [Public Gateway and HTTPS](./SRV-23-Public-Gateway-and-HTTPS.md) — the entry point
- [Public Site Operations](./SRV-25-Public-Site-Operations.md) — address updates, alerts, limits, backups
- [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md) and [CI/CD with GitHub Actions and GHCR](./SRV-14-CI-CD-GHCR.md) — the patterns reused here
- [Persistent Storage](./SRV-15-Persistent-Storage.md) — `local-path` volumes
- [SRV-24-TRBL](../troubleshooting/SRV-24-TRBL-Portfolio-Hosting.md) — troubleshooting for this doc
- [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-GitOps-and-ArgoCD.md) — companion Guides repository
