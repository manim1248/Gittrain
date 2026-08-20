# Lab 15 · Securing the Workflow Itself — Token, Supply Chain & Secrets

> **Confidential · Stalwart Learning**
> Module 15 · Guided lab · Session 3
> Companion: `guides/module-15-securing-the-workflow-itself.md` · Visualization: `module-15-visualization.html`

| | |
|---|---|
| **Objective** | Harden the workflows you've built across Session 3: least-privilege `GITHUB_TOKEN`, SHA-pinned actions, Dependabot for actions, secret hygiene — and a live demo of the **self-hosted runner trust boundary** (Setup 07) |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Setup 07 (self-hosted runner VM) recommended; Labs 11–14 workflows to audit. `ci-demo` private (required for the self-hosted runner part) |
| **Files you create** | `.github/dependabot.yml`, `permissions:` additions across existing workflows, a hardened `build-handoff.yml` |

---

## Step 1 · The three circles

Open visualization **15.1 The three concentric circles** — the token, the code it runs, the secrets it handles. This lab hardens all three on your actual workflows.

## Step 2 · Least-privilege GITHUB_TOKEN — audit & fix

Open visualization **15.2 permissions builder** and toggle the scenarios. Then audit your workflows from Labs 09–14:

1. For each workflow, add a **workflow-level floor**:

```yaml
permissions:
  contents: read
```

2. Where a job genuinely needs more, scope it **on the job** (Module 11 hand-off, Lab 13 prod):

```yaml
  update-manifests:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # only this job
```

3. Jobs doing OIDC (Lab 12) need `id-token: write` — on **that job**, not the whole workflow:

```yaml
  push:
    permissions:
      contents: read
      id-token: write
```

> **ADO mapping (guide §2):** ADO's `System.AccessToken` defaulted to full build-service scope. GitHub's model is opt-in per job — every `permissions:` you add is a privilege you didn't have to grant.

**Verify:** re-run the pipelines and check the run's job page shows the exact scopes you declared (run → job → "Set up job" step lists the token permissions).

## Step 3 · Pin third-party actions to full-length SHAs

Open visualization **15.3** and click each pin style. Then convert every `uses:` in `ci-demo` from tags to **immutable SHAs with version comments**:

```yaml
# BEFORE (mutable — v4 can be re-pointed by the owner)
- uses: actions/checkout@v4

# AFTER (immutable + readable)
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4
```

Find each SHA on the action's **README → Releases** (GitHub shows the pinned commit). Actions you'll likely pin:

| Action | SHA (example — fetch the real one for your version) |
|---|---|
| `actions/checkout` | `b4ffde65f46336ab88eb53be808477a3936bae11` (# v4) |
| `docker/build-push-action` | `4a13e500e55cf31b7a5d59a38ab2040ab0f42f56` (# v6) |
| `docker/login-action` | `9780b0c442fbb1117ed29e0efdff1e18412f7567` (# v3) |
| `azure/login` | fetch from https://github.com/Azure/login/releases |
| `azure/docker-login` | fetch from https://github.com/Azure/docker-login/releases |

> **Rule (guide §3):** full-length SHA, not short SHA, not tag. A tag can be moved; the SHA cannot. Add the version comment so humans still see intent.

## Step 4 · Dependabot keeps the SHAs current

Create `.github/dependabot.yml` so pinned actions stay patched:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      actions:
        patterns: ["*"]
```

Commit and push; within minutes Dependabot opens a PR bumping your pinned actions. Merge it — **then re-pin** the new SHAs if you want full audit immutability (or accept Dependabot's tag bump for velocity; document your choice).

## Step 5 · Secret hygiene — watch the leak

Open visualization **15.4** and run both the safe and the echo step. Then grep your own workflows for the anti-patterns:

```bash
rg -n 'echo "\$\{\{ secrets' .github/workflows/        # never echo a secret
rg -n 'secrets\..*\}' .github/workflows/ | wc -l        # count secret refs
```

Rules to apply (guide §4): secrets go into a step's `env:` and are consumed by tools — never interpolated into `run:` strings or echoed:

```yaml
# GOOD — secret lives in the step's env, never printed
- name: Deploy
  env:
    TOKEN: ${{ secrets.GITOPS_APP_TOKEN }}
  run: tool --token "$TOKEN" deploy

# BAD — interpolated into the command; greppable in run logs
- run: tool --token ${{ secrets.GITOPS_APP_TOKEN }} deploy
```

## Step 6 · The self-hosted runner trust boundary (Setup 07)

This is where Module 15's "runner risk model" (guide §5) becomes visceral:

1. Confirm your VM runner is **Idle** in repo → Settings → Actions → Runners (Setup 07 §4).
2. Create `.github/workflows/selfhosted-secure.yml`:

```yaml
name: Self-hosted secure job
on: workflow_dispatch
permissions:
  contents: read
jobs:
  on-vm:
    runs-on: [self-hosted, Linux]     # ← routes to your Azure VM
    steps:
      - name: Prove where this runs
        run: hostname && whoami && pwd
      - name: Runner trust note
        run: echo "Self-hosted = arbitrary code on MY host. Keep ci-demo private."
```

3. Run it and confirm the job executes on `ghactions-runner-01`.

**The security point (guide §5, Setup 07 §6):** any workflow change a `Write`-capable user merges can run arbitrary code on that VM with the repo's `GITHUB_TOKEN`. This is why:

- `ci-demo` must stay **private** (public + self-hosted = remote code execution on your VM).
- The runner should be in a **runner group** and gated by environment approval (Module 13) before it touches production hand-offs.

## Step 7 · The hardened skeleton

Assemble the full hardened pipeline (visualization **15.5**) — combine Lab 11's hand-off with everything above:

```yaml
name: Hardened Build & Hand-off
on:
  push:
    branches: [main]

permissions:
  contents: read                       # workflow-wide floor

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write                  # OIDC only here
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - uses: docker/build-push-action@4a13e500e55cf31b7a5d59a38ab2040ab0f42f56 # v6
        with:
          push: true
          tags: ${{ vars.ACR_NAME }}.azurecr.io/checkout-service:sha-${{ github.sha }}

  update-manifests:
    needs: build-push
    runs-on: ubuntu-latest
    permissions:
      contents: write                  # minimal, this job only
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4
        with:
          repository: <your-owner>/ci-gitops
          token: ${{ secrets.GITOPS_APP_TOKEN }}
      - name: Bump image reference (bot identity)
        run: |
          sed -i "s|image: .*checkout-service:.*|image: ${{ needs.build-push.outputs.image }}|" \
            apps/checkout-service/base/deployment.yaml
          git config user.name "ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git commit -am "ci: bump image"
          git push
```

## Step 8 · Validate with Copilot

> "Audit this workflow file for Module 15: enforce least-privilege permissions (workflow-level contents: read), pin every third-party action to a full-length SHA with a version comment, verify no run: interpolates a secret or untrusted input, and confirm there is no pull_request_target. Flag anything I missed."

Check Copilot didn't invent permissions, didn't leave a tag-pinned action, and didn't move `id-token: write` to workflow level.

---

## Expected outcome

- All workflows declare explicit least-privilege `permissions:` (job-scoped where needed).
- Every third-party action is SHA-pinned with a version comment; Dependabot is watching.
- A self-hosted job demonstrably ran on your Azure VM — and you can articulate why that means `ci-demo` stays private.
- The hardened skeleton replaces your Lab 11 workflow.

## Key takeaways

- **`permissions:` is opt-in, per-job** — the default should always be `contents: read`.
- **SHA-pinning + Dependabot = immutable and current** — the two halves of supply-chain hygiene.
- **A self-hosted runner is a trust boundary**, not a free CPU. Private repo, isolated host, gated access.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Workflow fails "Token permissions not sufficient" | You removed a needed scope — add it back on the *job* that needs it (Step 2) |
| Copilot/CI flags unknown action SHA | Use the exact full SHA from the action's Release page; 40 hex chars |
| Dependabot doesn't open a PR | Check Dependabot alerts settings; `directory: "/"` must match where `.github/dependabot.yml` lives |
| Self-hosted job never runs | `runs-on: [self-hosted, Linux]` must match the runner's labels; runner must show **Idle** (Setup 07 §4) |
| Secret appears in logs | You interpolated/echoed it — move to step `env:` (Step 5); rotate the leaked secret immediately |
| `pull_request_target` in a workflow | You shouldn't have it for these labs — it runs PR code with write privileges. Revert to `pull_request` (guide §2) |