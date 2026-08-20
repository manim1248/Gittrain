# Lab 11 · Deployment Integration — the Actions → GitOps Hand-off

> **Confidential · Stalwart Learning**
> Module 11 · Guided lab · Session 3
> Companion: `guides/module-11-deployment-integration.md` · Visualization: `module-11-visualization.html`

| | |
|---|---|
| **Objective** | Push an immutable image to **ACR**, then run a **controlled commit** job that updates the image reference in the **GitOps repo** (`ci-gitops`), and **watch Flux reconcile** the new image in AKS. Actions' job ends at the manifest update — Flux owns the rollout |
| **Time** | ~50 min (guided) |
| **Prerequisites** | Setups 08 (GitOps repo), 09 (AKS), 10 (Flux), 11 (ACR); 12 (OIDC) optional but recommended so the ACR push is keyless. Lab 09/10 CI on `ci-demo` green |
| **Files you create** | `.github/workflows/build-handoff.yml` (or extend your existing push workflow) |

---

## Step 1 · Revisit the boundary — what Actions owns, what it must not

Open visualization **11.1 Hand-off boundary** and click through all five steps. The rule that governs this lab:

> Actions writes to **git** (and the registry). Flux writes to the **cluster**. A `kubectl apply` step in this workflow would create two sources of truth — don't.

Your end state for this lab:

```
push → CI build → docker push ACR (sha-<7>) → commit manifests/ci-gitops ← boundary
                                                                    ↓
                                                       Flux reconciles AKS   (out of scope, observed)
```

## Step 2 · Inventory your current pipeline

You have a build/push workflow from Labs 09–10 pushing to **Docker Hub**. For this lab we extend the pattern to push to **ACR** and add the **manifest hand-off** job. Your inputs:

| Value | Where it comes from | Used as |
|---|---|---|
| `ACR_NAME` | Setup 11 | image prefix `<ACR_NAME>.azurecr.io` |
| `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | Setup 12 | OIDC login (Module 12) |
| `GITOPS_APP_TOKEN` | Setup 08 §4 | the bot commit to `ci-gitops` |
| `ci-gitops` | Setup 08 | the manifests repo |

> If Setup 12 is not complete, you can still do the **manifest hand-off** (Steps 4–5) by reusing your existing Docker Hub image reference — but the *keyless* ACR push needs OIDC. The instructor will demo the full path; follow with what you have.

## Step 3 · Build and push to ACR (keyless, via the Setup 12 trust)

Create `.github/workflows/build-handoff.yml`:

```yaml
name: Build & GitOps hand-off
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write          # for azure/login (Module 12)

jobs:
  build-push:
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.tag.outputs.image }}
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Compute immutable tag
        id: tag
        run: echo "image=${{ vars.ACR_NAME }}.azurecr.io/checkout-service:sha-${{ github.sha }}" >> "$GITHUB_OUTPUT"

      - name: Login to ACR (keyless)
        uses: azure/docker-login@v2
        with:
          login-server: ${{ vars.ACR_NAME }}.azurecr.io
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_ID }}

      - name: Build & push
        uses: docker/build-push-action@v6
        with:
          context: services/checkout
          file: services/checkout/Dockerfile
          push: true
          tags: ${{ steps.tag.outputs.image }}

      - name: Verify the tag exists (immutability check)
        run: az acr repository show-tags --name ${{ vars.ACR_NAME }} --repository checkout-service --top 3
```

> **What to notice (guide §2/§3):** the image is tagged by **`sha-<7>`** (immutable, traceable, rollback-able) — never `latest`. The push is **keyless** — no `ACR_PASSWORD` secret anywhere (that's the OIDC payoff, Module 12). The computed tag is exposed as a **job output** for the next job.

Push and confirm the tag appears:

```bash
git add . && git commit -m "lab11: build & push to ACR" && git push
# watch the run → check 'Verify the tag exists' step output
az acr repository show-tags --name $ACR_NAME --repository checkout-service -o table
```

## Step 4 · The hand-off job — update the manifest reference (controlled commit)

Add the second job. This is where **GitHub Actions stops**:

```yaml
  update-manifests:
    needs: build-push
    runs-on: ubuntu-latest
    permissions:
      contents: write              # minimal — only this job
    steps:
      - uses: actions/checkout@v4
        with:
          repository: <your-owner>/ci-gitops     # the manifests repo
          token: ${{ secrets.GITOPS_APP_TOKEN }} # the bot identity (Setup 08)

      - name: Update the image reference (base manifest)
        run: |
          sed -i "s|image: .*checkout-service:.*|image: ${{ needs.build-push.outputs.image }}|" \
            apps/checkout-service/base/deployment.yaml

      - name: Commit & push as the bot
        run: |
          git config user.name "ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git add apps/checkout-service/base/deployment.yaml
          git diff --cached --stat
          git commit -m "ci: bump checkout-service to ${{ needs.build-push.outputs.image }}"
          git push
```

Rules it follows (guide §4): **one atomic change** (the image line only), **bot identity** (`ci-bot`, not your PAT), **least privilege** (`contents: write` only here).

> **ADO mapping:** this job is the release step that *used* to run `kubectl apply`. Its GitHub equivalent is a git write — the deployment interface is the manifests repo, not the cluster (guide §1 table).

## Step 5 · Watch Flux do its job (the observable hand-off)

1. In a terminal, tail Flux's reconcile log (Setup 10 §5):

```bash
kubectl -n flux-system logs -f deployment/kustomize-controller
```

2. Back in the run, the `update-manifests` job commits `sha-<7>` to `ci-gitops`.
3. Within ~1 minute (Flux `--interval=1m`), the log shows a new **Revision** for `checkout-apps`.
4. Verify the rollout:

```bash
kubectl get deploy -n checkout -o wide
kubectl get pods -n checkout
```

The pod's image should now be `<ACR_NAME>.azurecr.io/checkout-service:sha-<7>` and running (not `ImagePullBackOff`).

> **Boundary recap:** you never touched the cluster from Actions. Actions updated a git repo; Flux turned that into a rollout. That's the whole module.

## Step 6 · Hand-off hygiene — score yourself

Open visualization **11.4 Hand-off hygiene** and flip the five rules. Against your workflow:

1. **Immutable tags** — `sha-<7>` ✓ (did you avoid `latest`?)
2. **One atomic change** — the `sed` touches one line ✓
3. **No kubectl** — none in the YAML ✓
4. **No trigger loops** — `ci-gitops` is a *separate repo*, so the manifest commit can't re-fire `build-handoff.yml` ✓ (contrast: if manifests lived in `ci-demo`, the bot's commit would re-trigger `on: push` — the pitfall in guide §4)
5. **Bot identity** — commit as `ci-bot` with the fine-grained `GITOPS_APP_TOKEN` ✓

## Step 7 · Validate with Copilot

Run this prompt (guide §6), then apply the Module 05 guardrails — read the output before merging:

> "Review my build-handoff.yml for the Module 11 hand-off boundary. Verify: the push job produces an immutable sha tag; the update-manifests job only writes to the ci-gitops repo with a bot token; there are no kubectl or cluster steps; permissions are least-privilege per job; and the manifest commit cannot re-trigger the workflow. Flag any violation."

Check Copilot didn't invent a `kubectl` step, didn't swap `GITOPS_APP_TOKEN` for `GITHUB_TOKEN`, and didn't drop the `permissions:` scoping.

---

## Expected outcome

- A `sha-<7>` image in ACR, pushed keylessly.
- A bot commit in `ci-gitops` bumping the image reference.
- Flux rolls the `checkout-service` Deployment to that image (observed, not performed by Actions).
- You can answer: *"Where does GitHub Actions' responsibility end, and what proves the hand-off worked?"*

## Key takeaways

- **The hand-off is a git write**, not a cluster action — keep it that way.
- **Immutable tags** are what make Flux diffing and rollbacks work.
- **Bot identity + separate manifests repo** prevent loops and keep the audit trail clean.

## Troubleshooting

| Symptom | Fix |
|---|---|
| ACR push fails `401 Unauthorized` | OIDC subject mismatch (Setup 12 §3) — verify `repo:<owner>/ci-demo:ref:refs/heads/main`, or the identity lacks `AcrPush` |
| `update-manifests` "permission denied" on push | `GITOPS_APP_TOKEN` must have **Contents: read+write** on `ci-gitops` only (Setup 08 §4) |
| Pod stays `ImagePullBackOff` | The image exists? (`az acr repository show-tags`); cluster kubelet has `AcrPull` (Setup 09 §5) or the pull path resolves `<ACR_NAME>.azurecr.io` |
| Flux doesn't roll within a minute | `flux get kustomizations` — is `checkout-apps` `Ready`? Did the `sed` actually change the line? `git -C ci-gitops show` the last commit |
| The manifest commit re-triggers a build | Manifests repo == app repo — move manifests to `ci-gitops` (Setup 08) and keep it separate |