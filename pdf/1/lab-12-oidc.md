# Lab 12 · Keyless Cloud Authentication — OIDC in the Workflow

> **Confidential · Stalwart Learning**
> Module 12 · Guided lab · Session 3
> Companion: `guides/module-12-keyless-cloud-authentication-oidc.md` · Visualization: `module-12-visualization.html`

| | |
|---|---|
| **Objective** | Use the OIDC trust from Setup 12 to run a **fully keyless** ACR push, then *abuse* the subject-claim model to see what trust boundaries actually protect. Prove there is **no cloud secret** in CI |
| **Time** | ~50 min (guided) |
| **Prerequisites** | Setup 12 (OIDC trust), Setup 11 (ACR), Lab 11 pushed an image via OIDC |
| **Files you create** | `.github/workflows/oidc-acr-push.yml`, a throwaway `subject-test.yml` |

---

## Step 1 · Review the trust model

Open visualization **12.1 Keyed vs keyless** and **12.2 The trust flow** — click through the five steps. The one sentence to internalise:

> **"The workflow is trusted because it *is* who it says it is"** — not because it holds a password. GitHub mints a signed `id_token`; Azure validates the signature **and the subject claim**; an access token valid for minutes is issued.

## Step 2 · Confirm the Setup 12 trust is live

Verify the federated credentials and RBAC you configured:

```bash
az ad app list --display-name gh-actions-oidc --query '[0].id' -o tsv | \
  xargs -I{} az ad app federated-credential list --id {} -o table
# expect: ci-main, cd-prod with the correct subjects

az role assignment list --assignee <app-id> --query '[].roleDefinitionName' -o tsv
# expect: AcrPush (AcrPull if you added it)
```

> **ADO mapping (guide §1 table):** this trust replaces the ACR **service connection** (username/password). ADO's *workload identity federation* is the same model — GitHub's is just native.

## Step 3 · The keyless push workflow

Create `.github/workflows/oidc-acr-push.yml`:

```yaml
name: Keyless ACR Push
on:
  push:
    branches: [main]
    paths: ['services/checkout/**']
  workflow_dispatch:

permissions:
  contents: read
  id-token: write                # ← the whole trick

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (keyless)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}        # identifiers, not credentials
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Login to ACR (no password — identity exchange)
        uses: azure/docker-login@v2
        with:
          login-server: ${{ vars.ACR_NAME }}.azurecr.io
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_ID }}          # the OIDC token, per-run

      - name: Build & push
        uses: docker/build-push-action@v6
        with:
          context: services/checkout
          file: services/checkout/Dockerfile
          push: true
          tags: ${{ vars.ACR_NAME }}.azurecr.io/checkout-service:sha-${{ github.sha }}
```

**The thing to prove:** search the run log — there is **no** `ACR_PASSWORD`, no service-principal key, no long-lived credential. The `password` field is populated at runtime by the token GitHub minted for *this run*.

Run it (`workflow_dispatch` or a checkout-service push) and verify the tag:

```bash
az acr repository show-tags --name $ACR_NAME --repository checkout-service -o table
```

## Step 4 · Break the trust — the subject-claim experiment

Open visualization **12.3 The subject claim** and click the four scenarios. Now reproduce the interesting one — **a PR cannot reach the `main` identity**.

1. On `ci-demo`, create a branch and change `oidc-acr-push.yml` so the `push` job also triggers on `pull_request` (temporarily):

```yaml
on:
  push:
    branches: [main]
  pull_request:                 # ← temporary, for the experiment only
```

2. Open a PR from that branch and watch the run.
3. **Expected result:** the `Azure login` step fails with `AADSTS700211` — because the run's `id_token` carries `ref:refs/pull/N/merge`, which matches **neither** `ci-main` (`refs/heads/main`) **nor** `cd-prod` (`environment:production`).

> **Why this matters (guide §5):** an attacker who can open a PR in your repo cannot impersonate your cloud identity. Narrow subjects are the security knob of the whole OIDC model — Setup 12 deliberately created *two* narrow credentials, not one broad `repo:*:ref:*`.

Revert the `pull_request` line before continuing.

## Step 5 · Narrower still — the environment subject (bridge to Module 13)

Setup 12 created `cd-prod` with subject `...environment:production`. Create a throwaway workflow that *declares* that environment and confirm it authenticates — this is what Module 13's `environment:` gating wires into OIDC:

```yaml
name: Prod identity test
on: workflow_dispatch
permissions:
  id-token: write
  contents: read
jobs:
  prod:
    runs-on: ubuntu-latest
    environment: production          # must exist or the job fails (Lab 13)
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az account show -o json
```

If `production` environment doesn't exist yet, create it now (repo → Settings → Environments → New environment → name `production`, no rules yet). When the run is green, you've proven: *environment name → subject claim → accepted by Azure*. That's the Module 12↔13 bridge.

## Step 6 · Troubleshooting practice

Open visualization **12.5 Troubleshooting** and deliberately break one thing, then fix it:

1. **Remove `id-token: write`** → run → "No OIDC token found" → add it back.
2. **Typo the subject** in a temporary federated credential → run → `AADSTS700211` → delete the typo'd credential.
3. **Remove the `AcrPush` role** (or assign it to the wrong scope) → push → `Authorization failed` → re-add.

Use the guide §7 decision tree to map each symptom to its fix.

## Step 7 · Validate with Copilot

> "Review my oidc-acr-push.yml: is id-token: write scoped correctly and not broader than needed? Are the Azure IDs in vars/secrets rather than hard-coded? For a federated credential restricted to repo:myorg/ci-demo:environment:production, what runs would fail, and why? Generate the az federated-credential create command for a main-branch CI identity with AcrPush."

Verify Copilot's answers against Setup 12: the subject strings must match exactly, and it should *not* suggest storing a client **secret** anywhere.

---

## Expected outcome

- Keyless ACR push green with **no cloud credential** in CI.
- You demonstrated (Step 4) that PR runs are blocked by subject claims.
- You demonstrated (Step 5) that an `environment:` job can reach its own identity — the OIDC + environments bridge used in Lab 13.

## Key takeaways

- **OIDC = trust by claims, not by password.** The subject is the security knob.
- `permissions: id-token: write` is the cost of admission; scope it to the job that needs it.
- Narrow subjects (`ref:`/`environment:`) are what make keyless *also* scoped.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `AADSTS700211` subject mismatch | Check the run's actual claims (Step 4) vs. federated subject — fix the credential subject in Entra |
| "No OIDC token found" | Missing `permissions: id-token: write` on the job |
| Audience error | Federated credentials must use `api://AzureADTokenExchange` |
| ACR push `Authorization failed` | `AcrPush` not granted, or granted to the wrong object/scope (Setup 12 §4) |
| PR run authenticates (bad!) | You used a broad subject like `repo:*:*` — narrow it (Setup 12 §3) |