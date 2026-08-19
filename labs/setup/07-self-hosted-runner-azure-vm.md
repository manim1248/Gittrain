# Setup 07 · Self-Hosted Runner — an Azure Linux VM

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration · Session 3

## Objective

Provision an **Azure Linux VM**, register it as a **self-hosted runner** for `ci-demo`, and prove it with a test job. Used by **Lab 15** to demonstrate the runner-trust boundary (Module 03 §6 recap, Module 15 §5 hardening) — and why self-hosted runners are a *trust boundary*, not a convenience.

> **Why now (Session 3)?** Module 15 covers securing the workflow *itself*. A self-hosted runner is the clearest demonstration of "code executing on a host you trust" — and the Module 15 rule *"never attach self-hosted runners to public repos"* lands better when you've just built one.

## 1. Create the VM

```bash
RG=ghactions-rg
VM_NAME=ghactions-runner
LOCATION=westeurope

az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Basic
```

> `Standard_B1s` (1 vCPU / 1 GiB) is enough to run a build step. Use `--size Standard_B2s` if the demo builds Docker images.

Open SSH:

```bash
az vm open-port --resource-group $RG --name $VM_NAME --port 22
```

## 2. Install the runner (SSH into the VM)

```bash
ssh azureuser@$(az vm show -g $RG -n $VM_NAME --query publicIps -o tsv)
```

On the VM:

```bash
# Docker (optional — only if your self-hosted job builds containers)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker azureuser

# Create a dedicated runner user (never run as root)
sudo useradd -m -s /bin/bash actions
sudo -u actions mkdir -p /home/actions/actions-runner
cd /home/actions/actions-runner

# Download the runner (version shown on github.com/<owner>/<repo>/settings/actions/runners/new)
curl -o actions-runner-linux-x64-2.323.0.tar.gz \
  -L https://github.com/actions/runner/releases/download/v2.323.0/actions-runner-linux-x64-2.323.0.tar.gz
tar xzf actions-runner-linux-x64-2.323.0.tar.gz
```

> Replace `2.323.0` with the version GitHub tells you. The exact URL is always shown on the **Runners → New self-hosted runner** page.

## 3. Get the registration token

github.com → `ci-demo` → **Settings → Actions → Runners → New self-hosted runner** → copy the **token** from the *Configure* section (it expires in 1 hour).

## 4. Configure and start the runner (still on the VM)

```bash
sudo -u actions ./config.sh \
  --url https://github.com/<your-owner>/ci-demo \
  --token <registration-token> \
  --name ghactions-runner-01 \
  --labels self-hosted \
  --work _work

sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

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

Run it manually → it should land on `ghactions-runner-01` (visible in the run's job header). You've just executed code on your own VM from a workflow.

## 6. Security notes — this is a trust boundary (Module 15 §5)

- **Never** attach this runner to a **public** repository — anyone with PR access can run arbitrary code on this VM (the `pull_request` fork-PR path). Keep `ci-demo` private.
- The runner holds the repo's `GITHUB_TOKEN` for the duration of each job — a compromised workflow = a compromised host.
- Use **runner groups** + environment approval (Module 13) before letting it run production hand-offs.
- Rotate/cleanup after the course: `sudo ./svc.sh uninstall`, delete the VM (`az vm delete`), and remove the runner in GitHub UI.

## 7. Verification checklist

- [ ] `az vm list -g $RG -o table` shows `ghactions-runner` running
- [ ] `ssh` into the VM; `sudo ./svc.sh status` shows the runner service **active**
- [ ] GitHub → `ci-demo` → Settings → Actions → Runners shows the VM **Idle** with label `self-hosted`
- [ ] The `selfhosted-test.yml` run executes on `ghactions-runner-01` with your `hostname`

## Troubleshooting

| Problem | Fix |
|---|---|
| `config.sh` fails "Authentication is required" | The registration token expired (1 h) — generate a new one on the Runners page |
| Runner shows **Offline** | `svc.sh start` failed — check `journalctl -u actions.runner.<owner>-<repo>.<name>.service` |
| Job never starts / waits forever | The workflow must use `runs-on: [self-hosted, Linux]` — a plain `ubuntu-latest` goes to GitHub-hosted runners |
| Job runs as `root` | You configured under root — recreate under the `actions` user (§2/§4) |
| Public repo warning | Self-hosted runners on public repos = RCE risk. Keep `ci-demo` private or use GitHub-hosted runners |