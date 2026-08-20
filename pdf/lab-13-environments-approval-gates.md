# Lab 13 · Environments & Approval Gates — Promote dev → staging → prod

> **Confidential · Stalwart Learning**
> Module 13 · Guided lab · Session 3
> Companion: `guides/module-13-environments-approval-gates.md` · Visualization: `module-13-visualization.html`

| | |
|---|---|
| **Objective** | Create `dev`/`staging`/`production` environments with protection rules, wire environment-scoped secrets/variables, build the promotion workflow (`needs:` chain + per-job `environment:`), and drive an approval through the UI. Bridge it to the OIDC `cd-prod` subject from Setup 12 |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Setups 08–12; Labs 11–12 pushed an ACR image keylessly. A **second GitHub account** (or a teammate) to act as required reviewer — the run actor cannot approve their own run |
| **Files you create** | `.github/workflows/promote.yml`, environments in repo settings |

---

## Step 1 · The mental model

Open visualization **13.1 What an environment bundles** — click all five rules. An environment is:

> **a named gate + config bundle** — protection rules (reviewers, branches, wait timer) **and** scoped secrets/variables **and** a deployment history. It's the GitHub analogue of an ADO stage + approvals + env-scoped variables (guide §1 table).

## Step 2 · Create the environments

repo `ci-demo` → **Settings → Environments → New environment**. Create three, each with rules:

| Environment | Required reviewers | Deployment branches | Notes |
|---|---|---|---|
| `dev` | none | `main` | frictionless — the pipeline's day-to-day |
| `staging` | your **second account** (or a teammate) | `main` | first human gate |
| `production` | your second account (or teammate) | `main` | the gate that matters |

**To add a reviewer you can actually approve with:** invite the second account as a **collaborator** (repo → Settings → Collaborators) *or* create an org + team and add both accounts to the team (Lab 14 does teams). For this lab, a collaborator is enough.

> **Self-approval rule (guide §3):** the account that *triggered* the run cannot approve it. If you trigger and approve with the same account, GitHub blocks it — hence the second account.

## Step 3 · Environment-scoped secrets & variables

Give each environment its own config so the **same workflow file** behaves differently per environment:

- `dev` → Variable `ENV_TARGET=dev`
- `staging` → Variable `ENV_TARGET=staging`
- `production` → Variable `ENV_TARGET=production`, and (as a demo) a **secret** `PROD_FLAG=controlled-rollout`

Set them: repo → **Settings → Environments → <env> → Environment variables / Environment secrets**.

> **ADO mapping (guide §2/§6):** environment variables/secrets ≈ ADO **environment-scoped variables**. Only a run targeting `production` sees `PROD_FLAG` — a `dev` run never does.

## Step 4 · The promotion workflow

Create `.github/workflows/promote.yml`:

```yaml
name: Promote
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.tag.outputs.image }}
    steps:
      - uses: actions/checkout@v4
      - id: tag
        run: echo "image=${{ vars.ACR_NAME }}.azurecr.io/checkout-service:sha-${{ github.sha }}" >> "$GITHUB_OUTPUT"

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - name: Target
        run: echo "deploying ${{ needs.build.outputs.image }} to ${{ vars.ENV_TARGET }}"
      - name: Hand off (Module 11) — bump dev overlay
        run: echo "bot-commit: set apps/checkout-service/overlays/dev image to ${{ needs.build.outputs.image }}"

  deploy-staging:
    needs: [build, deploy-dev]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Target
        run: echo "deploying to ${{ vars.ENV_TARGET }} (staging gate passed)"
      - name: Hand off — bump staging overlay
        run: echo "bot-commit: set apps/checkout-service/overlays/staging image"

  deploy-production:
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://${{ vars.ACR_NAME }}.azurecr.io   # demo URL, shown in the env UI
    steps:
      - name: Prod-only secret visible
        run: echo "PROD_FLAG is set: ${{ secrets.PROD_FLAG != '' }}"
      - name: Hand off — bump production overlay
        run: echo "bot-commit: set apps/checkout-service/overlays/production image"

  verify:
    needs: deploy-production
    runs-on: ubuntu-latest
    steps:
      - run: kubectl get deploy -n checkout -o wide   # requires kube context — see note
```

> **Structure rules (guide §5):** `environment:` is **per job**; `needs:` chains serialise promotion (production can't start until staging succeeded *and* was approved); the same immutable image flows through — only the manifest *reference* changes per environment. Keep `deploy-production`'s final hand-off as a **git write** (Module 11), not a `kubectl` — replace the demo `kubectl get` with a manifest commit for the full pattern.

## Step 5 · Run it and drive the approval

Open visualization **13.2 Approval flow** and **13.3 Promotion** first — then reproduce live:

1. Push to `main`.
2. **`deploy-dev`** runs instantly (no reviewers).
3. **`deploy-staging`** enters **Waiting for review**. Switch to the second account → Actions → find the run → **Review deployments** → **Approve**.
4. `deploy-staging` proceeds; **`deploy-production`** now waits — approve again with the second account.
5. Watch the environment pages accumulate **deployment history** (repo → Settings → Environments → `production`): commit, actor, status, URL.

> **What you just did:** the ADO "approve the release stage" moment, but as a first-class GitHub construct wired into the pipeline's `needs:` graph.

## Step 6 · Prove environment-scoping

- In a `deploy-dev` run, add a throwaway step `run: echo "$PROD_FLAG"` → it prints **empty** (the secret doesn't exist for `dev`).
- In a `deploy-production` run, the same step prints a masked `***` (it resolves).

That's environment isolation in action (Module 06 §4, now enforced per-environment).

## Step 7 · Bridge to OIDC (Setup 12's `cd-prod`)

The `production` environment job in Step 4 carries `environment:production` in its `id_token`. Combine it with the Azure login from Lab 12:

```yaml
      - name: Azure login with the production identity
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

Only this job's token matches the `cd-prod` subject (`...environment:production`) — a `dev` run fails to authenticate. Open visualization **13.5** to see the subject flow.

## Step 8 · Validate with Copilot

> "Review promote.yml: is environment: on every job, not just the workflow? Is the needs: chain dev→staging→production correct with the same image output? Are production's required reviewers and deployment branches documented? Generate a version where each environment's manifest hand-off bumps a different overlay path in ci-gitops, keeping it a git write with no kubectl."

Check Copilot: `environment:` on each job; `needs:` order right; no `kubectl apply`; production still gated.

---

## Expected outcome

- Three environments with protection rules; promotion blocked at `staging` and `production` until a reviewer approves.
- The same image promoted dev → staging → prod; per-environment variables/secrets resolve correctly.
- The `production` job can use the OIDC identity that only an `environment:production` run can reach.

## Key takeaways

- **`environment:` is a job-level key** — putting it at workflow level does nothing.
- **`needs:` + `environment:` = serialised, gated promotion.**
- **Environment secrets only resolve in their environment** — the same workflow file deploys everywhere safely.

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Waiting for review" forever | Approve from the **second account** — the run actor can't self-approve (Step 2) |
| `environment: production` at workflow level ignored | Move it onto the job (guide §9 pitfall 1) |
| `PROD_FLAG` empty in `deploy-dev` | Correct! It's scoped to `production` only (Step 6) |
| `deploy-production` never starts | `needs:` chain requires `deploy-staging` success *and* approval |
| `Azure login` fails only on the prod job | `cd-prod` subject mismatch — check `environment:production` in Setup 12 §3 |
| A `workflow_dispatch` from a feature branch reaches prod | You didn't set **deployment branches: main** on the environment (Step 2) |