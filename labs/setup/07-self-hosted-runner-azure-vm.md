# Setup 07 · Self-Hosted Runner — an Azure Linux VM

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Provision an **Azure Linux VM**, register it as a **self-hosted runner** for `ci-demo`, and prove it with a test job. Used by **Lab 15** to demonstrate the runner-trust boundary (Module 03 §6 recap, Module 15 §5 hardening) — and why self-hosted runners are a *trust boundary*, not a convenience.

> **Why now (Session 3)?** Module 15 covers securing the workflow *itself*. A self-hosted runner is the clearest demonstration of "code executing on a host you trust" — and the Module 15 rule *"never attach self-hosted runners to public repos"* lands better when you've just built one.

## 1. Create the VM

First generate an SSH keypair stored under `labs/setup/azure/` (the private key `id_ed25519` is gitignored — never commit it):

```bash
mkdir -p labs/setup/azure
ssh-keygen -t ed25519 -f labs/setup/azure/id_ed25519 -N "" -C "ghactions-runner azureuser"
```

```bash
RG=stalwart                # resource group holding the course VMs
VM_NAME=ghactions-runner   # vm name
LOCATION=eastus            # course VMs live here (different from the RG default region)
KEY_PUB=labs/setup/azure/id_ed25519.pub

az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --location $LOCATION \
  --image canonical:ubuntu-24_04-lts:server:latest \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --vnet-name vnet-eastus-1 \
  --subnet snet-eastus-1 \
  --ssh-key-value "$(cat $KEY_PUB)" \
  --public-ip-sku Standard
```

> Gotchas that bite here: `--location` is required — `az vm create` defaults to the resource group's region, but the course VMs are provisioned in `eastus`. Use `--public-ip-sku Standard` — the *Basic* SKU is quota-blocked in the training subscription (`IPv4BasicSkuPublicIpCountLimitReached`). Reuse the existing `vnet-eastus-1`/`snet-eastus-1` so a second VNet isn't created. Use `Standard_D2s_v3` — `Standard_B1s` (1 vCPU / 1 GiB) is too small once Docker is installed. Bump `VM_NAME` (e.g. `ghactions-runner-01` / `ghactions-runner-02`) to add more runners.

SSH (port 22) is already open — `az vm create` adds a `default-allow-ssh` NSG rule for Linux images. Only run `az vm open-port` if you closed it:

```bash
az vm open-port --resource-group $RG --name $VM_NAME --port 22
```

## 2. Install the runner (SSH into the VM)

```bash
ssh -i labs/setup/azure/id_ed25519 azureuser@$(az vm show -g $RG -n $VM_NAME --query publicIps -o tsv)
```

> `-i` selects the keypair from `labs/setup/azure/`. If you get `Permission denied (publickey)`, the VM was created with a different key — pass that key with `-i` instead.

On the VM:

```bash
# Create a dedicated runner user (never run as root)
sudo useradd -m -s /bin/bash actions

# Docker (needed if your self-hosted job builds containers)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker azureuser
sudo usermod -aG docker actions     # the runner runs as 'actions' — it needs docker access too

# Install the runner software (as root: azureuser can't cd into /home/actions)
sudo mkdir -p /home/actions/actions-runner
sudo curl -o /home/actions/actions-runner/actions-runner-linux-x64-2.336.0.tar.gz \
  -L https://github.com/actions/runner/releases/download/v2.336.0/actions-runner-linux-x64-2.336.0.tar.gz
sudo tar xzf /home/actions/actions-runner/actions-runner-linux-x64-2.336.0.tar.gz -C /home/actions/actions-runner
sudo chown -R actions:actions /home/actions/actions-runner
```

> Replace `2.336.0` with the version GitHub tells you. The exact URL is always shown on the **Runners → New self-hosted runner** page. Download, extract, and `chown` as `root` — `/home/actions` is owned by the `actions` user, so `cd /home/actions/actions-runner` as `azureuser` fails with `Permission denied`.

## 3. Get the registration token

github.com → `ci-demo` → **Settings → Actions → Runners → New self-hosted runner** → copy the **token** from the *Configure* section (it expires in 1 hour).

## 4. Configure and start the runner (still on the VM)

```bash
sudo -u actions bash -c 'cd /home/actions/actions-runner && ./config.sh \
  --url https://github.com/<your-owner>/ci-demo \
  --token <registration-token> \
  --name ghactions-runner-02 \
  --labels self-hosted \
  --work _work'

sudo bash -c 'cd /home/actions/actions-runner && ./svc.sh install actions'
sudo bash -c 'cd /home/actions/actions-runner && ./svc.sh start'
sudo bash -c 'cd /home/actions/actions-runner && ./svc.sh status'
```

> **`svc.sh install` needs the username as its second argument.** Without it the service runs as `$SUDO_USER` (here `azureuser`), which can't even read the `actions` home directory — the service dies instantly with `status=200/CHDIR ... Permission denied`. `install actions` makes systemd run the listener as the `actions` user. Every `svc.sh` call must run from the runner root, so wrap each in `sudo bash -c 'cd ... && ...'`.

Back in the GitHub UI, the runner shows **Idle** with the label `self-hosted`.

> **Token handling:** the registration token is short-lived and written into `.runner`/`.credentials` on the VM — it is not a workflow secret. Never echo it into logs (Module 15 §4).

## 5. Prove it with a test workflow

On `ci-demo`, create `.github/workflows/selfhosted-test.yml`:

```yaml
name: Self-hosted test
on: workflow_dispatch
jobs:
  probe:
    runs-on: [self-hosted, Linux]          # ← routes to the VM runner
    steps:
      - name: Where am I?
        run: hostname && uname -a && whoami
      - name: Runner labels
        run: echo "runner.os=${{ runner.os }} runner.arch=${{ runner.arch }}"
```

Run it manually → it should land on one of the self-hosted runners (`ghactions-runner-01` or `ghactions-runner-02` — both carry the same `self-hosted` label, visible in the run's job header). You've just executed code on your own VM from a workflow.

## 6. Security notes — this is a trust boundary (Module 15 §5)

- **Never** attach this runner to a **public** repository — anyone with PR access can run arbitrary code on this VM (the `pull_request` fork-PR path). Keep `ci-demo` private.
- The runner holds the repo's `GITHUB_TOKEN` for the duration of each job — a compromised workflow = a compromised host.
- Use **runner groups** + environment approval (Module 13) before letting it run production hand-offs.
- Rotate/cleanup after the course: `sudo ./svc.sh uninstall`, delete the VM (`az vm delete`), and remove the runner in GitHub UI.

## 7. Verification checklist

- [ ] `az vm list -g $RG -o table` shows `ghactions-runner-01`/`ghactions-runner-02` running
- [ ] `ssh` into the VM; `sudo ./svc.sh status` shows the runner service **active**
- [ ] GitHub → `ci-demo` → Settings → Actions → Runners shows the VMs **Idle** with label `self-hosted`
- [ ] The `selfhosted-test.yml` run executes on a self-hosted runner with your `hostname`

## Troubleshooting

| Problem | Fix |
|---|---|
| `config.sh` fails "Authentication is required" | The registration token expired (1 h) — generate a new one on the Runners page |
| Runner shows **Offline** | `svc.sh start` failed — check `journalctl -u actions.runner.<owner>-<repo>.<name>.service` |
| Job never starts / waits forever | The workflow must use `runs-on: [self-hosted, Linux]` — a plain `ubuntu-latest` goes to GitHub-hosted runners |
| Job runs as `root` | You configured under root — recreate under the `actions` user (§2/§4) |
| Service dies `status=200/CHDIR` / `Permission denied` on start | `svc.sh install` defaulted to `$SUDO_USER` — reinstall with `sudo ./svc.sh install actions` (§4) |
| `az vm create` fails `IPv4BasicSkuPublicIpCountLimitReached` | Use `--public-ip-sku Standard` (§1) |
| Public repo warning | Self-hosted runners on public repos = RCE risk. Keep `ci-demo` private or use GitHub-hosted runners |