# Setup 09 · AKS Cluster — Provision the Training Cluster

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Create a small, disposable **Azure Kubernetes Service (AKS)** cluster that Flux (Setup 10) will reconcile against. This is an **integration target** for the Session 3 hand-off labs — you need enough cluster to *observe* a rollout, not to operate production workloads.

> **Cost control:** a 1-node cluster is fine for the labs. AKS control plane is free; you pay only for nodes (~$30–60/mo depending on SKU). Delete the resource group after the course (cleanup at the end of this doc).

## 1. Prerequisites

- An **Azure subscription** (sandbox/dev subscription is fine — course outline §03) with `Contributor` or `Owner`.
- **Azure CLI** installed (`az version` works) and signed in: `az login`.
- Set your values once and reuse them:

```bash
RG=ghactions-rg
AKS_NAME=ghactions-aks
LOCATION=westeurope      # or any region with AKS capacity
```

> Replace `<your-values>` with real ones in every command below — the docs use placeholders only.

## 2. Create the resource group and cluster

```bash
az group create --name $RG --location $LOCATION

az aks create \
  --resource-group $RG \
  --name $AKS_NAME \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3 \
  --generate-ssh-keys \
  --yes
```

`--enable-cluster-autoscaler` keeps cost low (1–3 nodes). Creation takes **5–10 minutes**.

> **What you don't need here:** RBAC beyond the default, network plugins special-cased, or a service principal. The course's deployment path is GitOps (Flux), not `kubectl apply` from Actions — the cluster just needs to exist and be reachable.

## 3. Get credentials and verify

```bash
az aks get-credentials --resource-group $RG --name $AKS_NAME --overwrite-existing

kubectl get nodes
kubectl cluster-info
```

Expected: one `Ready` node. `kubectl` context is now `ghactions-aks`.

## 4. Create the namespace the manifests target

The GitOps repo (Setup 08) deploys into namespace `checkout`. Flux can create it from the manifests, but pre-creating it keeps reconcile logs clean:

```bash
kubectl create namespace checkout
```

## 5. Optional — connect your registry (needed before Module 12)

Skip if you're following the sequence and will do OIDC in Setup 12 / Lab 12. If you want the cluster to *pull* images from ACR **before** OIDC is wired, grant the cluster's kubelet identity `AcrPull`:

```bash
# Get the cluster's managed identity principal id
PRINCIPAL=$(az aks show --resource-group $RG --name $AKS_NAME --query identity.principalId -o tsv)

az role assignment create \
  --assignee $PRINCIPAL \
  --role AcrPull \
  --scope /subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.ContainerRegistry/registries/<acr-name>
```

> With OIDC (Lab 12) you can pull via the federated identity instead. Doing **both** is fine — `AcrPull` on kubelet is the standard AKS→ACR wiring.

## 6. Verification checklist

- [ ] `kubectl get nodes` shows one `Ready` node
- [ ] `kubectl get ns checkout` exists
- [ ] `az aks show -g $RG -n $AKS_NAME` returns identity.principalId (for the optional `AcrPull` step)

## Cleanup (after the course)

```bash
az group delete --name $RG --yes --no-wait
rm -f ~/.kube/config   # optional: drop the AKS context
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `az aks create` fails with quota/region error | Pick a different `LOCATION`, or reduce to `--node-count 1` without autoscaler |
| `kubectl` says "no configuration" | `az aks get-credentials` ran against the wrong subscription — check `az account show`, switch with `az account set --subscription <id>` |
| Node stays `NotReady` | Give it 2–3 minutes; `kubectl describe node` for the real reason (usually image pull or resource pressure) |
| Cost worries | `kubectl scale deploy` nothing yet — stop the autoscaler and `az aks scale --node-count 0` between sessions is not supported on system pools; instead delete/stop the cluster when idle |