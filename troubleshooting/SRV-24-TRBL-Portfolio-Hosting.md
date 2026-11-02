---
tags: [homelab, project, kubernetes, mongodb, argocd, github-actions, ghcr, gitops, troubleshooting]
---

# Troubleshooting — Portfolio Hosting

> Companion to [Portfolio Hosting](../server/SRV-24-Portfolio-Hosting.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [Pods in `ErrImagePull` before the first build](#pods-in-errimagepull-before-the-first-build) | [Step 3](../server/SRV-24-Portfolio-Hosting.md#step-3--manifests-and-the-argocd-application) |
| [Pasted commands garbled by the terminal](#pasted-commands-garbled-by-the-terminal) | [Step 2](../server/SRV-24-Portfolio-Hosting.md#step-2--namespace-and-secrets) |
| [GHCR packages are private by default](#ghcr-packages-are-private-by-default) | [Step 1](../server/SRV-24-Portfolio-Hosting.md#step-1--images-and-the-build-workflow) |
| [Image tag reset to `bootstrap`](#image-tag-reset-to-bootstrap) | [Step 3](../server/SRV-24-Portfolio-Hosting.md#step-3--manifests-and-the-argocd-application) |
| [ArgoCD shows `OutOfSync` after a successful sync](#argocd-shows-outofsync-after-a-successful-sync) | [Step 3](../server/SRV-24-Portfolio-Hosting.md#step-3--manifests-and-the-argocd-application) |
| [The Atlas cluster no longer existed](#the-atlas-cluster-no-longer-existed) | [Step 5](../server/SRV-24-Portfolio-Hosting.md#step-5--load-the-data) |
| [An empty answer from `/api/health` during a rollout](#an-empty-answer-from-apihealth-during-a-rollout) | [Verification](../server/SRV-24-Portfolio-Hosting.md#verification) |

## Pods in `ErrImagePull` before the first build

**Symptom:** right after the Application was applied, `api` and `web` were `ErrImagePull` while `mongo-0` was `ContainerCreating`.

**Root cause:** the manifests start with the placeholder tag `bootstrap`, which no registry holds. Expected: the images do not exist before the first workflow run.

**Fix:** none needed beyond the build. After the first run, the workflow's second job replaced both tags and the pods started.

## Pasted commands garbled by the terminal

**Symptom:** `kubectl create namespace portfolio` answered `kubectl: command not found` with stray `[200~` and `[201~` characters in the echoed line, and the following label command failed with `invalid label value "...restricted~"`. The secret commands then failed with `namespaces "portfolio" not found`.

**Root cause:** the terminal's bracketed-paste markers ended up inside the pasted text, so the first two commands were corrupted and the namespace was never created. Nothing was created by the failing secret commands.

**Fix:** the commands were run again, one at a time (`kubectl create namespace portfolio`, then the label command), and the secret block afterwards.

**Check added to the procedure:** give long manifests and multi-step blocks as small, separate commands, and read the echoed line before trusting the result.

## GHCR packages are private by default

**Symptom:** none observed, because the step was done before the first rollout. New GHCR packages are created **private**.

**Root cause (avoided):** the cluster pulls without credentials, so a private package would give `ErrImagePull` with an authorisation error.

**Fix:** both packages were made public right after the first build: GitHub profile → Packages → `render-resume-web` and `render-resume-api` → Package settings → Danger zone → Change visibility. The pods retried by themselves.

## Image tag reset to `bootstrap`

**Symptom:** the Slack alert `KubePodNotReady` (`portfolio/api-…`, non-ready for more than 15 minutes). `kubectl -n portfolio get pods` showed a new `api-…` pod in `ImagePullBackOff` next to the old one, which kept serving; `describe pod` ended with `render-resume-api:bootstrap: not found`.

**Root cause:** `api.yaml` had been edited in a scratch copy (to change the allowed origins) and copied over the file in `homelab-gitops`. The scratch copy still held the placeholder tag, so the copy **overwrote the commit hash that the workflow had written**. `web.yaml` was not copied and kept its tag.

**Fix:** the real tag was read from the other manifest and written back in a branch and pull request:

```
TAG=$(grep -o 'render-resume-web:[0-9a-f]*' apps/portfolio/web.yaml | cut -d: -f2); echo "$TAG"
sed -i "s|render-resume-api:bootstrap|render-resume-api:$TAG|" apps/portfolio/api.yaml
git diff | grep '^[+-] '
```

After the merge ArgoCD rolled out the right image and the alert resolved.

**Check added to the procedure:** never copy `web.yaml` or `api.yaml` from a scratch copy back into the repository; edit the file in the repository, or pull first and change only the line in question.

## ArgoCD shows `OutOfSync` after a successful sync

**Symptom:** the Application tree showed `OutOfSync` (yellow arrows) on the Mongo StatefulSet and the three `HTTPRoute` objects while `Last sync` was `Sync OK` to the same commit and health was `Healthy`.

**Root cause:** the API server fills in default fields (StatefulSet claim templates, route match defaults) that are not in the files; ArgoCD's default comparison reads the difference as drift.

**Fix:** compare on the server side:

```
kubectl -n argocd annotate application portfolio argocd.argoproj.io/compare-options=ServerSideDiff=true --overwrite
kubectl -n argocd annotate application portfolio argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd get application portfolio        # Synced, Healthy
```

The annotation is also in the Application manifest, so a recreated Application keeps it.

## The Atlas cluster no longer existed

**Symptom:** `mongodump` from a container on the PC stopped at option parsing: `lookup _mongodb._tcp.<cluster>.mongodb.net ... no such host`. `Resolve-DnsName -Type SRV _mongodb._tcp.<cluster>.mongodb.net` answered `DNS name does not exist`.

**Root cause:** the cluster had been deleted; an existing cluster, even a paused one, still resolves. The data could not be dumped.

**What was tried:** the development Mongo volume (`docker volume`, `myweb_mongo-data`, 303 MB) was copied and opened with a throwaway container: the `myweb` database had empty `projects` and `skills` collections, so it never held the real content.

**Fix:** the JSON exports kept in the application repository (`server/db/R&R.skills.json`, `projects.json`, from October 2025) were imported with `mongoimport --jsonArray`. Content edited in Atlas after that date is lost.

**Check added to the procedure:** keep a dump of the content in the repository or in the backup; a managed database that is shut down takes its data with it. The nightly backup now includes the `portfolio` database ([SRV-25](../server/SRV-25-Public-Site-Operations.md)).

## An empty answer from `/api/health` during a rollout

**Symptom:** `curl -s https://<DOMAIN>/api/health` printed nothing right after a rollout, while the page returned `200`.

**Root cause:** the request hit the moment the API pod was replaced. `curl -s` hides the error.

**Fix:** repeating the request with `curl -si` showed `HTTP/2 200` and `{"ok":true,"env":"production"}`.

**Check added to the procedure:** use `-i` when a silent `curl` shows nothing, to see the status line.

## Related

- [Portfolio Hosting](../server/SRV-24-Portfolio-Hosting.md) — the working procedure
- [SRV-14-TRBL](./SRV-14-TRBL-CI-CD-GHCR.md) — an earlier empty image tag in a manifest
- [SRV-13-TRBL](./SRV-13-TRBL-ArgoCD-GitOps.md) — ArgoCD and GitHub incidents
- [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-GitOps-and-ArgoCD.md) — companion Guides repository
