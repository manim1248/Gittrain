# Setup 12 · OIDC — Keyless Trust Between GitHub Actions and Azure

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Establish the **one-time OIDC trust** between GitHub Actions and Azure that replaces long-lived cloud credentials. You'll create an identity Azure trusts, add **federated credentials** with scoped subject claims, grant RBAC, and store the identity's IDs as GitHub secrets/variables. Lab 12 then *uses* this trust for keyless ACR pushes.

> **Mental model (Module 12 guide §2):** GitHub mints a signed `id_token` whose *subject* says *which repo + ref/environment*. Azure accepts it **only** if it matches a federated credential you configure here. No secret ever sits in CI.

## 1. Prerequisites

- `az` CLI signed in; `RG=ghactions-rg` and `ACR_NAME` from Setup 11.
- Repo `ci-demo` (the app repo from Session 1) to store secrets/variables.

## 2. Create the identity Azure trusts

Two options — pick one. The lab uses an **App Registration (service principal)**:

```bash
RG=ghactions-rg
ACR_NAME=<your-acr>

# Option A — App Registration (service principal)
APP_NAME=gh-actions-oidc
az ad app create --display-name $APP_NAME > /tmp/app.json
APP_ID=$(az ad app list --display-name $APP_NAME --query '[0].appId' -o tsv)

# Option B — User-Assigned Managed Identity (alternative)
# az identity create --name gh-actions-oidc --resource-group $RG
```

> `client-id`/`appId` is **public metadata**, not a secret — you'll store it as a GitHub *variable*.

## 3. Add federated credentials — the subject claims

Each federated credential defines *one allowed subject*. We create two, scoping trust to exactly what the labs need:

| Name | Subject | Purpose |
|---|---|---|
| `ci-main` | `repo:<owner>/ci-demo:ref:refs/heads/main` | CI pushes to ACR from `main` |
| `cd-prod` | `repo:<owner>/ci-demo:environment:production` | the **production** environment job (Module 13) |

```bash
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name":"ci-main",
    "issuer":"https://token.actions.githubusercontent.com",
    "subject":"repo:<your-owner>/ci-demo:ref:refs/heads/main",
    "audiences":["api://AzureADTokenExchange"]
  }'

az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name":"cd-prod",
    "issuer":"https://token.actions.githubusercontent.com",
    "subject":"repo:<your-owner>/ci-demo:environment:production",
    "audiences":["api://AzureADTokenExchange"]
  }'
```

Replace `<your-owner>` (your GitHub username/org) in both subjects.

> **Why two?** Module 12's security rule is "the narrower the subject, the safer". `ci-main` lets CI push images; `cd-prod` is only reachable by a run gated to the `production` environment — the Module 12↔13 bridge. A PR run matches *neither* and gets nothing.

## 4. Grant RBAC — what the identity may do

The same identity pushes to ACR (CI) and, via the prod subject, could manage registry artifacts:

```bash
az role assignment create \
  --assignee $APP_ID \
  --role AcrPush \
  --scope /subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME

# Optional: let the same identity pull in the cluster (read-only role)
az role assignment create \
  --assignee $APP_ID \
  --role AcrPull \
  --scope /subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME
```

## 5. Store the identity IDs in GitHub

On `ci-demo` (app repo):

- **Variables** → *Settings → Secrets and variables → Actions → Variables*:
  - `AZURE_TENANT_ID` = your Entra tenant id (`az account show --query tenantId -o tsv`)
  - `AZURE_SUBSCRIPTION_ID` = `az account show --query id -o tsv`
  - `ACR_NAME` = your registry name
- **Secrets** → *New repository secret*: `AZURE_CLIENT_ID` = `$APP_ID`

> Client IDs are identifiers, not credentials — `vars` for the two IDs *plus* `ACR_NAME`, and `AZURE_CLIENT_ID` as a secret because the `azure/login` examples in this course keep the three IDs together. Either split is defensible; the labs read them as shown.

## 6. Verify the trust with a one-off run (before Lab 12)

Create a throwaway workflow on a branch, run it, and confirm the id_token is exchanged:

```yaml
name: OIDC smoke test
on: workflow_dispatch
permissions:
  id-token: write
  contents: read
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Azure login (keyless)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az account show -o json   # proves the exchange worked
```

Expect a green run showing your subscription. A `AADSTS700211` failure means a subject mismatch — check the subject strings in §3.

## 7. Verification checklist

- [ ] `az ad app show --id $APP_ID` shows two federated credentials (`ci-main`, `cd-prod`)
- [ ] `az role assignment list --assignee $APP_ID` includes `AcrPush` (and optional `AcrPull`)
- [ ] GitHub vars `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `ACR_NAME` and secret `AZURE_CLIENT_ID` set on `ci-demo`
- [ ] The OIDC smoke-test run is green with **no** client secret stored anywhere

## Troubleshooting

| Problem | Fix |
|---|---|
| `AADSTS700211` (subject mismatch) | Compare the token's actual `repo:`/`environment:` claims with the federated subject. A PR run has `ref:refs/pull/N/merge` — matches nothing |
| `audience` error | All three credentials must use `"audiences":["api://AzureADTokenExchange"]` |
| "No OIDC token found" | The job needs `permissions: id-token: write` |
| `Authorization failed` on ACR push | `AcrPush` assigned to the **wrong object** or the role hasn't propagated (~30s); re-check `az role assignment list --assignee $APP_ID` |
| Works on `main`, fails in Lab 13 prod job | The `cd-prod` subject requires `environment: production` — confirm the job declares it (Lab 13) |