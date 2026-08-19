# Setup 08 · GitOps Repository — Provision the Manifests Repo

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Create the **GitOps (manifests) repository** that GitHub Actions hands off to: a dedicated repo holding the Kubernetes manifests for `checkout-service`, with per-environment overlays, a CODEOWNERS file, and the **bot identity** the workflows will use to commit image-reference bumps (Module 11).

> **Scope note (from the outline):** this is the *integration point only*. Flux reconciliation and AKS deployment mechanics are out of scope for the course — you only need this repo so the Module 11 hand-off has somewhere to write, and so Flux (Setup 10) can observe the change.

## 1. Create the manifests repository

1. github.com → **New repository** → name it **`ci-gitops`** (private).
2. Do **not** add a README yet (Flux bootstrap in Setup 10 works cleanest on an empty repo).
3. This repo is *separate* from `ci-demo` — that separation is what prevents trigger loops (Module 11 §4).

## 2. Lay out the manifest structure

We use a small Kustomize-style layout — one directory per service, with a base + per-environment overlay. The image reference lives in `base` and each environment can pin its own (Module 13 re-points these per environment):

```
ci-gitops/
└── apps/
    └── checkout-service/
        ├── base/
        │   ├── kustomization.yaml
        │   └── deployment.yaml
        └── overlays/
            ├── dev/       (kustomization.yaml)
            ├── staging/   (kustomization.yaml)
            └── production/(kustomization.yaml)
```

Create `apps/checkout-service/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-service
  namespace: checkout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: checkout-service
  template:
    metadata:
      labels:
        app: checkout-service
    spec:
      containers:
        - name: checkout-service
          image: <ACR_NAME>.azurecr.io/checkout-service:sha-initial   # ← the reference Actions updates
          imagePullPolicy: IfNotPresent
```

Create `apps/checkout-service/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
```

Each overlay (`dev`, `staging`, `production`) is a minimal `kustomization.yaml` that extends base:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
```

> **Why the placeholder `sha-initial`?** Flux will fail to pull an image that doesn't exist. `sha-initial` keeps the Deployment valid until Module 11 replaces it with a real `sha-<7>` tag. Update it with your real `ACR_NAME` from Setup 11.

## 3. Add a CODEOWNERS file (Module 14 uses it)

Create `.github/CODEOWNERS` in `ci-gitops`:

```text
# Production manifests are owned by the reviewers of production changes
/manifests/** @<your-github-username>
/apps/** @<your-github-username>
```

> In the classroom you'll map this to real teams (`@sre-team` etc.); for a solo account, your own handle stands in until Lab 14 exercises teams.

## 4. Create the bot identity — `GITOPS_APP_TOKEN`

The workflows commit manifest bumps as a **bot**, not as you (Module 11 §4). Use a **fine-grained PAT**:

1. github.com → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
2. Repository access: **Only select repositories** → `ci-gitops`.
3. Permissions → **Repository permissions**: *Contents → Read and write*.
4. Expiration: 90 days (training). Generate and **copy it once** into your password manager.
5. On `ci-demo` → **Settings → Secrets and variables → Actions → New repository secret**: name `GITOPS_APP_TOKEN`, value the token above.

> **Why fine-grained?** A classic PAT with full scope could write to every repo you can see. The fine-grained token is limited to `ci-gitops` — the least privilege your hand-off job needs (Module 15).

## 5. Verification checklist

- [ ] `ci-gitops` exists (private) with `apps/checkout-service/base/deployment.yaml` + `kustomization.yaml` and three overlays
- [ ] The image reference in `base` matches your `<ACR_NAME>.azurecr.io` (placeholder `sha-initial`)
- [ ] `.github/CODEOWNERS` exists
- [ ] `GITOPS_APP_TOKEN` secret exists on `ci-demo` and can push to `ci-gitops` (test with a manual commit, then revert)

## Troubleshooting

| Problem | Fix |
|---|---|
| Flux can't reconcile the Deployment | Image reference must be `<ACR_NAME>.azurecr.io/checkout-service:<tag>` exactly — Flux reports the failed pull in `flux get kustomizations` (Setup 10) |
| CI bot commit "permission denied" | The fine-grained token must have **Contents: read+write** on `ci-gitops` only; re-scope it |
| `kustomization.yaml` missing | Flux (and `kubectl -k`) fail with "unable to find one of 'kustomization.yaml'..." — every overlay dir needs one |