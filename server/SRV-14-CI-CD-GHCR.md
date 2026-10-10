---
tags: [homelab, project, kubernetes, ci-cd, github-actions, ghcr, gitops]
---

# CI/CD with GitHub Actions and GHCR

> Status: 🟢 **Working.** A push to the `my-site` repository builds the image in GitHub Actions and publishes it to GHCR; the cluster pulls it from there, and `my-site` no longer depends on an image imported by hand into one node. The image tag in `homelab-gitops` is updated by the workflow itself, so a push to `my-site` reaches the cluster with no manual step. 🟡 Open: see [Open items](#open-items).

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
9. [Step 7 — Replace the token with deploy keys](#step-7--replace-the-token-with-deploy-keys)
10. [Step 8 — Automatic tag update](#step-8--automatic-tag-update)
11. [Verification](#verification)
12. [Open items](#open-items)
13. [Files](#files)
14. [Related](#related)

Problems hit during this work are recorded separately in [SRV-14-TRBL](../troubleshooting/SRV-14-TRBL-CI-CD-GHCR.md).

## Components

| Component | Where | Role |
|---|---|---|
| `my-site` repository | `github.com/MrSandwick/my-site` (public) | Site source: `index.html`, `Dockerfile`, and the workflow |
| `build-and-push` workflow | `.github/workflows/build.yml` in `my-site` | On every push to `main`: build the image, push it to GHCR |
| GHCR | `ghcr.io/mrsandwick/my-site` | Image registry; every node pulls from it |
| `homelab-gitops` repository | `github.com/MrSandwick/homelab-gitops` | Manifests; the image reference lives in `apps/my-site/my-site.yaml` |
| ArgoCD | Namespace `argocd` | Applies the manifest when the tag in git changes |
| Deploy keys | One per repository on GitHub | SSH access for pushes from the nodes and from the workflow ([Step 7](#step-7--replace-the-token-with-deploy-keys), [Step 8](#step-8--automatic-tag-update)) |
| `GITOPS_DEPLOY_KEY` | Actions secret in `my-site` | Private half of a write key for `homelab-gitops`, used by the workflow |

Two repositories, two questions: `my-site` answers *what is the site*, `homelab-gitops` answers *what is running in the cluster*. Code changes often; cluster state should change deliberately. Concepts and registry alternatives: [Guide: Container Registries and GHCR](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Container-Registries-and-GHCR.md).

## End state

```
git push ──→ my-site (GitHub) ──→ GitHub Actions ──→ GHCR
                                  │ builds the image   ghcr.io/mrsandwick/my-site:<COMMIT_SHA>
                                  │                            │ pulled by any node
                                  └─ commits the new tag ──→ homelab-gitops ──→ ArgoCD ──→ Deployment my-site (default)
```

Until [Step 8](#step-8--automatic-tag-update) the tag edit in `homelab-gitops` was manual. The whole path from a push to a running pod is now automatic.

## Step 1 — Repositories and token

On github.com, create an empty public repository `my-site` — no README, otherwise the first push conflicts with it.

Pushes use the existing fine-grained Personal Access Token from [SRV-13 Step 5](./SRV-13-ArgoCD-GitOps.md#step-5--push-access). Three things on the token, checked before the first push:

| Setting | Value | Why |
|---|---|---|
| Repository access | `homelab-gitops` **and** `my-site` | A token only works on repositories it lists |
| Contents | Read and write | Push commits |
| Workflows | Read and write | GitHub rejects a push that adds a file under `.github/workflows/` without it |

> **Later change:** this token was replaced by per-repository SSH deploy keys and deleted — see [Step 7](#step-7--replace-the-token-with-deploy-keys). It was used for Steps 2–6.

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

## Step 7 — Replace the token with deploy keys

The fine-grained token from [SRV-13 Step 5](./SRV-13-ArgoCD-GitOps.md#step-5--push-access) had a 30-day lifetime and covered every repository it listed. It was replaced by SSH **deploy keys**, one per repository, and then deleted.

| | Fine-grained token | Deploy key |
|---|---|---|
| Scope | Every repository on its list | One repository |
| Lifetime | Expires (30 days here) | Does not expire |
| Permissions | A list to get right (Contents, Workflows) | Read, or read and write |
| Where it lives | A value shown once | A key pair on the node |

A deploy key belongs to exactly one repository, so each repository gets its own pair. The private half never leaves the node, the public half is added in the repository's **Settings → Deploy keys**, visible only to its administrators. The keys have an empty passphrase (`-N ""`), because the nodes are headless; the limit is that each key only opens one repository.

**On `<HOSTNAME>` — `homelab-gitops`:**

```
ssh-keygen -t ed25519 -f ~/.ssh/homelab-gitops-deploy -C "homelab-gitops deploy key" -N ""
cat ~/.ssh/homelab-gitops-deploy.pub
```

The whole `.pub` line is added to `homelab-gitops` → Settings → Deploy keys, title `<HOSTNAME>`, **Allow write access** on. Then:

```
cat >> ~/.ssh/config <<'EOF'
Host github-gitops
  HostName github.com
  User git
  IdentityFile ~/.ssh/homelab-gitops-deploy
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
ssh -T git@github-gitops
cd ~/homelab-gitops
git remote set-url origin git@github-gitops:MrSandwick/homelab-gitops.git
git fetch
```

`Host github-gitops` is an alias: it makes SSH present this key for this repository only (`IdentitiesOnly yes` stops it trying other keys). Hence the remote URL uses `github-gitops:` instead of `github.com:`. Expected answer of `ssh -T`: `Hi MrSandwick/homelab-gitops! You've successfully authenticated, but GitHub does not provide shell access.`

At the first connection SSH asks to confirm the host key. GitHub's published ED25519 fingerprint is `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`; it matched what was shown. A push to `homelab-gitops` with the key was verified with an empty commit (`test: verify deploy key write access`).

**On `<WORKER_HOSTNAME>` — `my-site`:** the same steps with a separate key (`~/.ssh/my-site-deploy`), title `<WORKER_HOSTNAME>`, host alias `github-mysite`, and in `~/my-site`:

```
git remote set-url origin git@github-mysite:MrSandwick/my-site.git
```

ArgoCD is unaffected: it reads the public `homelab-gitops` repository over HTTPS.

Once both repositories worked over SSH, the fine-grained token was deleted (Settings → Developer settings → Fine-grained tokens).

## Step 8 — Automatic tag update

Until now the new commit hash was written into `homelab-gitops` by hand after every build. The workflow now does it. It needs write access to `homelab-gitops`, which is a third key pair, separate from the node keys:

1. On `<HOSTNAME>`: `ssh-keygen -t ed25519 -f ~/.ssh/actions-gitops-deploy -C "github-actions my-site" -N ""`.
2. The `.pub` line is added to `homelab-gitops` → Deploy keys, title `github-actions (my-site)`, write access on.
3. The **private** file's whole content (`-----BEGIN` to `-----END`) is stored as the Actions secret `GITOPS_DEPLOY_KEY` in `my-site` (Settings → Secrets and variables → Actions).
4. The private file is then deleted from the node: `shred -u ~/.ssh/actions-gitops-deploy`. It is needed only by GitHub.

Two steps are appended to the workflow of [Step 3](#step-3--build-workflow), on `<WORKER_HOSTNAME>`:

```yaml
      - name: Check out homelab-gitops
        uses: actions/checkout@v4
        with:
          repository: MrSandwick/homelab-gitops
          ssh-key: ${{ secrets.GITOPS_DEPLOY_KEY }}
          path: gitops

      - name: Update image tag in homelab-gitops
        run: |
          cd gitops
          sed -i "s|image: ghcr.io/mrsandwick/my-site:.*|image: ghcr.io/mrsandwick/my-site:${{ github.sha }}|" apps/my-site/my-site.yaml
          grep -q "my-site:${{ github.sha }}" apps/my-site/my-site.yaml
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git commit -am "my-site: deploy ${{ github.sha }}"
          git push
```

| Part | Effect |
|---|---|
| `ssh-key: ${{ secrets.GITOPS_DEPLOY_KEY }}` | Checks out `homelab-gitops` over SSH with the write key; the repository's own `permissions:` block (which limits the automatic token) is not involved |
| `sed … ${{ github.sha }}` | Writes the hash of the commit that was just built into the manifest |
| `grep -q …` | Fails the step if the replacement did not happen, so a missing or wrong tag is never pushed (compare [SRV-14-TRBL](../troubleshooting/SRV-14-TRBL-CI-CD-GHCR.md#empty-tag-in-the-manifest-argocd-comparisonerror)) |
| `git commit` / `git push` as `github-actions[bot]` | The change is an ordinary commit in `homelab-gitops`, visible in its history |

The bot's push does not trigger the `my-site` workflow (different repository), so there is no loop. ArgoCD then rolls the new tag out as in Step 6.

`homelab-gitops` is now written by two parties. Run `git pull` in `~/homelab-gitops` on `<HOSTNAME>` before editing a manifest by hand.

## Verification

| Check | Result |
|---|---|
| Actions run `build-and-push` on commit `9c0db67` | Success, 35 s |
| `kubectl get pods -l app=my-site -o wide` | Two pods `Running` on `<WORKER_HOSTNAME>` |
| Workflow with the tag-update steps ([Step 8](#step-8--automatic-tag-update)) | Green. `homelab-gitops` received `my-site: deploy 69bc2a98…` from `github-actions[bot]` (commit `6ef6d2e`) |
| `kubectl get application my-site -n argocd` | `Synced` / `Healthy` |
| Image on the pods after the automated rollout | `ghcr.io/mrsandwick/my-site:69bc2a98bf0653d95142e70977991f290f4ce71c` on both pods |
| `ssh -T` and `git fetch` with the deploy keys | Authenticated; no password prompt |
| `kubectl describe pod -l app=my-site \| grep -i image:` | `ghcr.io/mrsandwick/my-site:<COMMIT_SHA>` on both pods |

## Open items

- ⬜ **End-to-end content change not yet exercised.** The chain was verified by the workflow change itself; editing `index.html`, pushing, and seeing the new page at `/site` with no manual command remains to be shown.
- ⬜ **Local image not removed.** `docker.io/library/my-site:v1` is still in the worker's containerd and no longer used: `sudo ctr -n k8s.io images rm docker.io/library/my-site:v1`.

## Files

| File | Location | Contents |
|---|---|---|
| `.github/workflows/build.yml` | `my-site` repository | Build and push workflow |
| `index.html`, `Dockerfile` | `my-site` repository (source on `<WORKER_HOSTNAME>`) | Site and image build |
| `apps/my-site/my-site.yaml` | `homelab-gitops` repository | Deployment + Service with the registry image |
| `~/.ssh/homelab-gitops-deploy`, `.pub`; `~/.ssh/config` | `<HOSTNAME>` | Deploy key and host alias for `homelab-gitops` |
| `~/.ssh/my-site-deploy`, `.pub`; `~/.ssh/config` | `<WORKER_HOSTNAME>` | Deploy key and host alias for `my-site` |
| `GITOPS_DEPLOY_KEY` | Actions secret in `my-site` | Private half of the workflow's write key |

## Related

- [SRV-14-TRBL](../troubleshooting/SRV-14-TRBL-CI-CD-GHCR.md) — troubleshooting for this doc
- [ArgoCD and GitOps](./SRV-13-ArgoCD-GitOps.md) — the GitOps loop this extends
- [First Real Workload](./SRV-10-First-Real-Workload.md) — the node-local image this replaces
- [Guide: GitHub Authentication — Tokens vs Deploy Keys](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/host/Guide-GitHub-Authentication-Tokens-vs-Deploy-Keys.md) — companion Guides repository
- [Persistent Storage](./SRV-15-Persistent-Storage.md) — the next step
- [Guide: Repository Secrets (GitHub Actions)](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-GitHub-Actions-Repository-Secrets.md) — companion Guides repository
- [Guide: Container Registries and GHCR](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Container-Registries-and-GHCR.md) — companion Guides repository
- [Guide: Local Container Images Without a Registry](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Local-Container-Images-Without-a-Registry.md) — companion Guides repository
- [Guide: GitOps and ArgoCD](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/platform/Guide-GitOps-and-ArgoCD.md) — companion Guides repository
