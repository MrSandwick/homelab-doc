---
tags: [homelab, project, kubernetes, ci-cd, github-actions, ghcr, gitops, troubleshooting]
---

# Troubleshooting — CI/CD with GitHub Actions and GHCR

> Companion to [CI/CD with GitHub Actions and GHCR](../server/SRV-14-CI-CD-GHCR.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Step |
|---|---|
| [Same directory name on two nodes](#same-directory-name-on-two-nodes) | [Step 6](../server/SRV-14-CI-CD-GHCR.md#step-6--switch-the-cluster-to-the-registry-image) |
| [Token not scoped for the new repository and workflows](#token-not-scoped-for-the-new-repository-and-workflows) | [Step 1](../server/SRV-14-CI-CD-GHCR.md#step-1--repositories-and-token) |
| [`git commit` fails: author identity unknown](#git-commit-fails-author-identity-unknown) | [Step 2](../server/SRV-14-CI-CD-GHCR.md#step-2--git-identity) |
| [Empty tag in the manifest: ArgoCD ComparisonError](#empty-tag-in-the-manifest-argocd-comparisonerror) | [Step 6](../server/SRV-14-CI-CD-GHCR.md#step-6--switch-the-cluster-to-the-registry-image) |

## Same directory name on two nodes

**What happened:** the site source was copied from the control plane to the worker for the image build ([SRV-10](../server/SRV-10-First-Real-Workload.md)), so both nodes have a `~/my-site`. The git repository was created on the worker; the copy on the control plane was never a repository. Commands written against `~/my-site` behaved differently depending on the node they were pasted into, and this led directly to the incident [below](#empty-tag-in-the-manifest-argocd-comparisonerror).

**Fix:** every command block in the procedure now states the node it runs on (`<HOSTNAME>` for the control plane and `kubectl`, `<WORKER_HOSTNAME>` for the site source and git push). Values that must not depend on a local clone, such as the commit hash, are read from GitHub rather than from a directory that may exist on only one node.

## Token not scoped for the new repository and workflows

**What was found:** before the first push, the token's page showed `homelab-gitops` and `my-site` under repository access but only **Metadata (read)** and **Contents (read and write)** under permissions. **Workflows** was missing.

**Root cause:** the token was created for `homelab-gitops` ([SRV-13](../server/SRV-13-ArgoCD-GitOps.md#step-5--push-access)). GitHub treats a push that creates or changes a file under `.github/workflows/` as a separate permission from pushing code, so a token that can push everything else is still refused for the workflow file. The refusal looks like the earlier 403 ([SRV-13-TRBL](./SRV-13-TRBL-ArgoCD-GitOps.md#github-push-403-with-a-fine-grained-token)), which makes it easy to misdiagnose.

**Fix:** Settings → Developer settings → Fine-grained tokens → the token → **Edit** → add **Workflows: Read and write**, with `my-site` in the repository list. Editing permissions keeps the token's value. Caught before the push, so no failed attempt occurred. The token was later replaced by deploy keys ([SRV-14 Step 7](../server/SRV-14-CI-CD-GHCR.md#step-7--replace-the-token-with-deploy-keys)), which have no permission list to get wrong.

The token's value cannot be viewed again after creation; if it is not saved, **Regenerate token** is the only way back, which keeps permissions and repositories but invalidates the old value.

## `git commit` fails: author identity unknown

**Symptom:** on the worker, `git commit` refused to run with *Author identity unknown*.

**Root cause:** git had never been configured for the user on that node. The identity lives in the user's `~/.gitconfig` (`git config --global`), per user and per machine; the control plane had one, the worker did not.

**Fix:**

```
git config --global user.name "MrSandwick"
git config --global user.email "<GitHub noreply address>"
```

Check with `git config --global user.name`, `git config --global user.email`, or `git config --list --show-origin`. The noreply address from GitHub → Settings → Emails keeps the real address out of public commits. This is separate from the token, which is only asked for at `git push`.

## Empty tag in the manifest: ArgoCD ComparisonError

**Symptom:** ArgoCD showed the application in `ComparisonError`:

```
Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = FailedPrecondition desc = Failed to unmarshal "my-site.yaml": failed to unmarshal manifest: error converting YAML to JSON: yaml: line 17: mapping values are not allowed in this context
```

**Root cause:** the image line in the manifest read `image: ghcr.io/mrsandwick/my-site:` — a trailing colon with no tag. The `sed` command had inserted a shell variable, `SHA=$(git -C ~/my-site rev-parse HEAD)`, which was empty because `~/my-site` on the control plane is not a git repository (see [above](#same-directory-name-on-two-nodes)); `git` failed, but nothing checked the result, so the empty value was substituted silently. A bare colon at the end of the value makes the YAML parser read a nested mapping, hence *mapping values are not allowed in this context*. The broken file was then committed and pushed (`91b83ef`), so GitHub, and therefore ArgoCD, held the broken version.

**Impact:** none on the running site. ArgoCD could not parse the new manifest, so it applied nothing and the existing pods kept running on the old image.

**Misdiagnosis along the way:** after the first fix, the same error was still shown with the note `(cached)`. It was assumed to be ArgoCD's cached failure from the earlier attempt. It was not: the corrected file had been edited in the working tree but never committed. `git status` showed `Changes not staged for commit` on `apps/my-site/my-site.yaml`, and `git log` showed the broken commit still at the top. ArgoCD reads GitHub, not the working directory, so it was correctly still reporting the broken version.

**Fix:**

```
cd ~/homelab-gitops/apps/my-site
SHA=$(git ls-remote https://github.com/MrSandwick/my-site.git HEAD | cut -f1)
echo $SHA
sed -i "s|image: ghcr.io/mrsandwick/my-site:.*|image: ghcr.io/mrsandwick/my-site:$SHA|" my-site.yaml
grep -n "image:" my-site.yaml
kubectl apply --dry-run=client -f my-site.yaml
git add my-site.yaml
git commit -m "my-site: fix image tag"
git push
```

then force ArgoCD to re-read the repository (UI: the arrow beside **Refresh** → **Hard Refresh**, or `kubectl annotate application my-site -n argocd argocd.argoproj.io/refresh=hard --overwrite`). Two pods `Running` on `<WORKER_HOSTNAME>` followed, with the `ghcr.io` image.

**Checks added to the procedure:**

- `echo $SHA` before use, and stop if it is empty or does not start with the hash shown in Actions.
- `kubectl apply --dry-run=client -f <file>` before every manifest commit: it parses the file and applies nothing, and would have caught this before the push.
- After a push, confirm it landed: `git status` reports `up to date with 'origin/main'` with nothing unstaged, and `git log -1 --oneline` shows the intended commit.
- When ArgoCD reports a manifest error, compare what is on GitHub with the working tree before suspecting a stale cache.

## Related

- [CI/CD with GitHub Actions and GHCR](../server/SRV-14-CI-CD-GHCR.md) — the working procedure
- [SRV-13-TRBL](./SRV-13-TRBL-ArgoCD-GitOps.md) — earlier token and scheduling incidents
- [Guide: Container Registries and GHCR](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/containers/Guide-Container-Registries-and-GHCR.md) — companion Guides repository
