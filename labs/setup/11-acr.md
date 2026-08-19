# Setup 11 · ACR — Provision the Container Registry

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Create an **Azure Container Registry (ACR)** that Session 3 workflows push to and AKS pulls from. We deliberately create it **without** the admin credentials — the labs authenticate **keylessly via OIDC** (Setup 12, Lab 12), which is the whole point of Module 12.

> **Design choice:** a classic training setup enables ACR's admin user (username + password). We don't — the outline's Module 12 message is *"replace long-lived cloud credentials"*. OIDC to ACR push is the production pattern you're here to learn; keep it that way.

## 1. Prerequisites

- Azure subscription from Setup 09, `az` CLI signed in.
- Reuse `RG=ghactions-rg` from Setup 09.

## 2. Create the registry

```bash
RG=ghactions-rg
ACR_NAME=<pick-a-globally-unique-name, e.g. ghactionsacr<yourinitials>>
LOCATION=westeurope

az acr create \
  --resource-group $RG \
  --name $ACR_NAME \
  --sku Basic \
  --admin-enabled false
```

- **SKU:** `Basic` is the cheapest and fine for labs (10 GiB storage, 50 concurrent pulls).
- **`--admin-enabled false`:** no static username/password. Authentication comes from identity (Setup 12).

## 3. Confirm admin is off (and see the registry)

```bash
az acr show --name $ACR_NAME --query 'adminUserEnabled' -o tsv     # false
az acr list --resource-group $RG -o table
```

If the instructor prefers a fallback for quick demos, you *may* enable admin **temporarily** and rotate later — but every lab that matters uses OIDC:

```bash
# OPTIONAL fallback only — remember to re-disable before production thinking
az acr update --name $ACR_NAME --admin-enabled true
```

## 4. Tag convention the labs expect

The workflows (Labs 11–12) push to:

```
$ACR_NAME.azurecr.io/checkout-service:sha-<7-sha>
```

The `sha-<7>` tag is the **immutable** reference Flux reconciles on (Module 09 §4 tagging strategy). Write `$ACR_NAME` down — you'll use it in GitHub `vars` (Lab 12) and in the GitOps repo (Setup 08).

## 5. Verification checklist

- [ ] `az acr show -n $ACR_NAME -g $RG --query adminUserEnabled -o tsv` → `false`
- [ ] You know your `$ACR_NAME` (globally unique, lowercase)
- [ ] (Optional) AKS kubelet has `AcrPull` from Setup 09 §5, **or** you'll wire pull via OIDC identity in Setup 12

## Troubleshooting

| Problem | Fix |
|---|---|
| `az acr create` name taken | ACR names are globally unique — append initials/random, e.g. `ghactionsacrjs` |
| `docker push` to ACR fails with 401 | You have no identity yet — that's expected until Setup 12/Lab 12. Don't fall back to admin creds unless the instructor approves |
| Pull from AKS `ErrImagePull` with 401 | kubelet lacks `AcrPull` (Setup 09 §5) or the pull identity isn't granted `AcrPull` yet |
| Need to check what's in the registry | `az acr repository list --name $ACR_NAME` (works without docker) |