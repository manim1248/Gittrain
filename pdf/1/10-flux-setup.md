# Setup 10 · Flux — Bootstrap GitOps on the Training Cluster

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Install **Flux** into the AKS cluster (Setup 09) so it watches the GitOps repo (Setup 08) and reconciles the manifests. This gives you a working observation point for the Module 11 hand-off: when Actions updates an image reference, you'll *see* Flux roll the deployment.

> **Scope note (from the outline):** Flux reconciliation mechanics are **out of scope** — the course covers only the hand-off boundary. You need Flux running so the labs can verify "the hand-off worked", not to learn Flux. Follow this doc mechanically; Module 11 lab tells you what to watch.

## 1. Prerequisites

- `ci-gitops` repo (Setup 08), `ghactions-aks` cluster (Setup 09), `kubectl` context set.
- **flux CLI** installed:

```bash
# macOS (Homebrew)
brew install fluxcd/tap/flux

# or via curl script
curl -s https://fluxcd.io/install.sh | sudo bash

flux --version   # expect v2.x
```

## 2. Bootstrap Flux with GitHub

`flux bootstrap` installs the controllers in the cluster *and* writes the Flux manifests into the GitOps repo — git becomes the source of truth for Flux itself.

```bash
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=ci-gitops \
  --branch=main \
  --path=./clusters/ghactions \
  --personal
```

Flags explained:

| Flag | Value | What it does |
|---|---|---|
| `--owner` | your GitHub username | repo owner (org for teams) |
| `--repository` | `ci-gitops` | the GitOps repo from Setup 08 |
| `--path` | `./clusters/ghactions` | where Flux stores its own manifests in that repo |
| `--personal` | — | authenticates with your PAT (prompted) and skips org-level config |

Bootstrap takes ~2–4 minutes and ends with a git commit to `ci-gitops`.

## 3. Verify Flux is healthy

```bash
flux check
flux get kustomizations
```

Expected: `flux-system` kustomization with `READY=True` and `Applied`.

## 4. Point Flux at the checkout-service manifests

Create a `Kustomization` that tells Flux to apply everything under `apps/`:

```bash
flux create kustomization checkout-apps \
  --target-namespace=checkout \
  --source=flux-system \
  --path=./apps \
  --prune=true \
  --interval=1m
```

Then confirm it reconciles the manifests you committed in Setup 08:

```bash
flux get kustomizations
kubectl get deploy -n checkout
```

Expected: `checkout-service` Deployment appears in namespace `checkout` (its image `...:sha-initial` may show `ImagePullBackOff` until a real image exists — that's expected and fine).

## 5. Watch the reconcile (what Module 11 lab verifies)

Open a second terminal and tail Flux's event log while you push a manifest change later in Lab 11:

```bash
kubectl -n flux-system logs -f deployment/kustomize-controller
```

When Actions bumps the image reference in `ci-gitops`, you'll see a `Revision` change and a new rollout.

## 6. Verification checklist

- [ ] `flux check` passes
- [ ] `flux get kustomizations` shows `flux-system` and `checkout-apps` both `Ready`
- [ ] `kubectl get deploy -n checkout` shows the `checkout-service` Deployment from the GitOps repo
- [ ] `ci-gitops` contains a new commit from bootstrap under `clusters/ghactions`

## Cleanup

```bash
flux uninstall --silent
az group delete --name ghats-rg --yes --no-wait   # if no longer needed
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `bootstrap github` auth error | The PAT must have `repo` scope; for orgs use `--owner=<org>` and a PAT with `org:read` + repo access. You may need `GITHUB_TOKEN=<token>` exported |
| `flux check` reports pre-reqs missing | `kubectl get crds` — if Flux was already installed once, run `flux uninstall --silent` and bootstrap again |
| `checkout-apps` Kustomization `Ready=False, Source not ready` | The `--path=./apps` must exist in `ci-gitops` with at least one `kustomization.yaml` (Setup 08) |
| `checkout-service` pod `ErrImagePull` | Expected until a real ACR image exists. Set `imagePullPolicy: IfNotPresent` is not the issue — pull auth is (Setup 09 §5 / Lab 12 OIDC) |
| Deploys only happen after a delay | Default `--interval=1m` means up to ~1 min lag between the Actions commit and the rollout — that's the Flux polling model |