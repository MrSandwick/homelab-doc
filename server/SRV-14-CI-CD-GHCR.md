---
tags: [homelab, project, kubernetes, ci-cd, github-actions, ghcr, gitops]
---

# CI/CD with GitHub Actions and GHCR

> Status: 🟢 **Working.** A push to the `my-site` repository builds the image in GitHub Actions and publishes it to GHCR; the cluster pulls it from there, and `my-site` no longer depends on an image imported by hand into one node. 🟡 Open: the image tag in `homelab-gitops` is still updated by hand — see [Open items](#open-items).

Picks up after [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md), closing its open item about a registry-hosted image. Each step says which node it runs on; two nodes hold a directory called `~/my-site`, and only one of them is a git repository.

## Contents

1. [Components](#components)
2. [End state](#end-state)
3. [Step 1 — Repositories and token](#step-1--repositories-and-token)
4. [Step 2 — Git identity](#step-2--git-identity)
5. [Step 3 — Build workflow](#step-3--build-workflow)
6. [Step 4 — Push and check the build](#step-4--push-and-check-the-build)
7. [Step 5 — Make the package public](#step-5--make-the-package-public)
8. [Step 6 — Switch the cluster to the registry image](#step-6--switch-the-cluster-to-the-registry-image)
9. [Verification](#verification)
10. [Open items](#open-items)
11. [Files](#files)
12. [Related](#related)

Problems hit during this work are recorded separately in [SRV-14-TRBL](../troubleshooting/SRV-14-TRBL-CI-CD-GHCR.md).

## Components

| Component | Where | Role |
|---|---|---|
| `my-site` repository | `github.com/MrSandwick/my-site` (public) | Site source: `index.html`, `Dockerfile`, and the workflow |
| `build-and-push` workflow | `.github/workflows/build.yml` in `my-site` | On every push to `main`: build the image, push it to GHCR |
| GHCR | `ghcr.io/mrsandwick/my-site` | Image registry; every node pulls from it |
| `homelab-gitops` repository | `github.com/MrSandwick/homelab-gitops` | Manifests; the image reference lives in `apps/my-site/my-site.yaml` |
| ArgoCD | Namespace `argocd` | Applies the manifest when the tag in git changes |

Two repositories, two questions: `my-site` answers *what is the site*, `homelab-gitops` answers *what is running in the cluster*. Code changes often; cluster state should change deliberately. Concepts and registry alternatives: [Guide: Container Registries and GHCR](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Container-Registries-and-GHCR.md).

## End state

```
git push ──→ my-site (GitHub) ──→ GitHub Actions ──→ GHCR
                                  builds the image    ghcr.io/mrsandwick/my-site:<COMMIT_SHA>
                                                              │ pulled by any node
homelab-gitops (GitHub) ── tag edited by hand ──→ ArgoCD ──→ Deployment my-site (default)
```

The manual link is the tag edit between GHCR and `homelab-gitops`; everything else is automatic.

## Step 1 — Repositories and token

On github.com, create an empty public repository `my-site` — no README, otherwise the first push conflicts with it.

Pushes use the existing fine-grained Personal Access Token from [SRV-13 Step 5](./SRV-13-ArgoCD-GitOps.md#step-5--push-access). Three things on the token, checked before the first push:

| Setting | Value | Why |
|---|---|---|
| Repository access | `homelab-gitops` **and** `my-site` | A token only works on repositories it lists |
| Contents | Read and write | Push commits |
| Workflows | Read and write | GitHub rejects a push that adds a file under `.github/workflows/` without it |

Editing a token's permissions keeps its value. A token's value is shown only once, at creation; if it was not saved, the only option is **Regenerate token**, which keeps permissions and repositories but invalidates the old value.

## Step 2 — Git identity

Run on `<WORKER_HOSTNAME>`, where the site source and image build live. Git identity is per user, in `~/.gitconfig`; a freshly installed node has none and `git commit` fails with *Author identity unknown*.

```
git config --global user.name "MrSandwick"
git config --global user.email "<GitHub noreply address>"
```

The noreply address (GitHub → Settings → Emails → *Keep my email addresses private*) keeps the real address out of public commits. Check what is set:

```
git config --global user.name
git config --global user.email
git config --list --show-origin
```

## Step 3 — Build workflow

On `<WORKER_HOSTNAME>`:

```
cd ~/my-site
mkdir -p .github/workflows
nano .github/workflows/build.yml
```

```yaml
name: build-and-push
on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/mrsandwick/my-site:${{ github.sha }}
```

| Line | Effect |
|---|---|
| `on: push: branches: [main]` | Runs on every push to `main` |
| `permissions: packages: write` | Lets the job's automatic token publish to GHCR |
| `password: ${{ secrets.GITHUB_TOKEN }}` | A short-lived token GitHub issues for each run; no secret is created or stored by hand |
| `tags: …:${{ github.sha }}` | Tag = the full commit hash. Every build gets a unique name, so the manifest changes when the version changes |
| Image name in lowercase | GHCR rejects uppercase; the account name `MrSandwick` is written `mrsandwick` |

**Decision — commit hash, not `latest`:** with a fixed tag the manifest never changes, so ArgoCD has nothing to detect, and a node holding a cached `latest` keeps running the old build. With a unique tag, git records exactly which version is deployed, and rolling back is reverting a commit.

The `${{ … }}` expressions are filled in by GitHub and are not edited.

## Step 4 — Push and check the build

On `<WORKER_HOSTNAME>`:

```
git init -b main
git add .
git commit -m "Add static site, Dockerfile, and CI workflow"
git remote add origin https://github.com/MrSandwick/my-site.git
git push -u origin main
```

When git asks for a password, the token is entered (not displayed while typed).

On GitHub: `my-site` → **Actions**. The run `build-and-push` finished green in 35 seconds (the build job 31 seconds, the image build 10); the pushed commit was `9c0db67`. The package `my-site` then appears under the account's **Packages**.

## Step 5 — Make the package public

New GHCR packages are created private. Package settings → Danger Zone → **Change visibility** → **Public**. A public image is pulled without credentials, which suits a static site with no secrets; a private image would need an `imagePullSecret` in the cluster.

## Step 6 — Switch the cluster to the registry image

Run on `<HOSTNAME>`, where `~/homelab-gitops` and `kubectl` are. The tag is the **full** 40-character hash of the commit that was built. It is read from GitHub, not from a local clone, because `~/my-site` on this node is not a git repository:

```
cd ~/homelab-gitops/apps/my-site
SHA=$(git ls-remote https://github.com/MrSandwick/my-site.git HEAD | cut -f1)
echo $SHA
```

`echo` must print 40 characters beginning with the short hash shown in Actions. **If it prints nothing, stop** — an empty value breaks the manifest, see [SRV-14-TRBL](../troubleshooting/SRV-14-TRBL-CI-CD-GHCR.md#empty-tag-in-the-manifest-argocd-comparisonerror).

```
sed -i "s|image: docker.io/library/my-site:v1|image: ghcr.io/mrsandwick/my-site:$SHA|" my-site.yaml
sed -i '/imagePullPolicy: Never/d' my-site.yaml
sed -i '/nodeSelector:/,+1d' my-site.yaml
grep -n "image:" my-site.yaml
kubectl apply --dry-run=client -f my-site.yaml
git diff
```

| Change | Why |
|---|---|
| `image:` → `ghcr.io/mrsandwick/my-site:<COMMIT_SHA>` | Pull the published build instead of the node-local import |
| Remove `imagePullPolicy: Never` | The default policy pulls an image that is not present on the node |
| Remove `nodeSelector` | The pin existed only because the image lived on one node ([SRV-10](./SRV-10-First-Real-Workload.md)) |

`kubectl apply --dry-run=client` checks that the file parses and applies nothing; it is run before every manifest push. `git diff` should show one replaced line and three removed.

**Decision — drop the `nodeSelector`:** pods still land on `<WORKER_HOSTNAME>`, because the control plane is tainted and the laptop is cordoned ([SRV-13 Step 2](./SRV-13-ArgoCD-GitOps.md#step-2--pod-placement)). That placement is implicit, though: uncordoning the laptop would let the pods move there. **Alternative not used:** a `nodeSelector` on a node label (`kubectl label node <WORKER_HOSTNAME> workload=apps`, then `workload: apps`) — explicit, and independent of hostnames.

Then:

```
git add apps/my-site/my-site.yaml
git commit -m "my-site: switch to GHCR image, drop nodeSelector"
git push
```

ArgoCD polls about every three minutes; **Refresh** in the UI forces it.

## Verification

| Check | Result |
|---|---|
| Actions run `build-and-push` on commit `9c0db67` | Success, 35 s |
| `kubectl get pods -l app=my-site -o wide` | Two pods `Running` on `<WORKER_HOSTNAME>` |
| `kubectl describe pod -l app=my-site \| grep -i image:` | `ghcr.io/mrsandwick/my-site:<COMMIT_SHA>` on both pods |

## Open items

- ⬜ **Tag update is manual.** After each build the new hash has to be written into `homelab-gitops`. Two ways to automate it: a final step in the workflow that commits the new tag to `homelab-gitops` (needs a token stored as a secret in `my-site`), or ArgoCD Image Updater running in the cluster. The first is simpler and visible in the workflow file; the second adds another component to the cluster.
- ⬜ **Local image not removed.** `docker.io/library/my-site:v1` is still in the worker's containerd and no longer used: `sudo ctr -n k8s.io images rm docker.io/library/my-site:v1`.
- ⬜ **Token lifetime.** The fine-grained token expires 30 days after creation. An SSH deploy key for the headless server does not expire ([SRV-13 Step 5](./SRV-13-ArgoCD-GitOps.md#step-5--push-access)).
- ⬜ Items carried over from [SRV-13](./SRV-13-ArgoCD-GitOps.md#open-items): admin password rotation, laptop in the Ansible inventory.

## Files

| File | Location | Contents |
|---|---|---|
| `.github/workflows/build.yml` | `my-site` repository | Build and push workflow |
| `index.html`, `Dockerfile` | `my-site` repository (source on `<WORKER_HOSTNAME>`) | Site and image build |
| `apps/my-site/my-site.yaml` | `homelab-gitops` repository | Deployment + Service with the registry image |

## Related

- [SRV-14-TRBL](../troubleshooting/SRV-14-TRBL-CI-CD-GHCR.md) — troubleshooting for this doc
- [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md) — the GitOps loop this extends
- [First Real Workload](./SRV-10-First-Real-Workload.md) — the node-local image this replaces
- [Guide: Container Registries and GHCR](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Container-Registries-and-GHCR.md) — companion Guides repository
- [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Local-Container-Images-Without-a-Registry.md) — companion Guides repository
- [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-GitOps-and-ArgoCD.md) — companion Guides repository
