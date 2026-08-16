# Lab 01 · GitHub Actions Architecture — your first workflow

> **Confidential · Stalwart Learning**
> Module 01 · Guided lab · Session 1
> Companion: `guides/module-01-github-actions-architecture.md` · Visualization: `module-01-visualization.html`

| | |
|---|---|
| **Objective** | Observe the execution model (event → workflow → job → step → runner) live on your own repo |
| **Time** | ~35 min (guided, demo-driven) |
| **Prerequisites** | `labs/setup/` steps 1–3 complete (`ci-demo` repo, Actions enabled, VS Code connected) |
| **Files you create** | `.github/workflows/ci.yml` |

---

## Pre-flight (2 min)

- [ ] Signed in to `github.com`; `ci-demo` repo exists (Setup 2)
- [ ] VS Code open on `ci-demo`; GitHub Actions + YAML extensions show no errors (Setup 3)
- [ ] `git status` is clean

---

## Step 1 · Create the workflow file

In VS Code, create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

env:
  NODE_VERSION: '20'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repository
        uses: actions/checkout@v4
      - name: Print the event that started this run
        run: echo "event=${{ github.event_name }} ref=${{ github.ref }} sha=${{ github.sha }}"
      - name: Install and build
        run: |
          echo "Using Node ${{ env.NODE_VERSION }} on ${{ runner.os }}"
          npm init -y >/dev/null 2>&1 && npm pkg set scripts.build="echo build-ok"
          npm run build
```

Save. The VS Code GitHub Actions extension should validate it with no red squiggles.

## Step 2 · Push and watch a run

```bash
git add .github/workflows/ci.yml
git commit -m "lab01: first workflow"
git push
```

1. Open the **Actions** tab → the `CI` run triggered by `push`.
2. Click the run, then the `build` job, then expand each step:
   - **Check out the repository** — note the checkout log (the runner just cloned your repo).
   - **Print the event…** — see `event=push ref=refs/heads/main sha=<your sha>`.
   - **Install and build** — the shell commands.
3. Note the **runner label** `ubuntu-latest` in the job header and the **Status** badges.

**Watch for the execution model you just saw:** an event (push) loaded the workflow, the job ran on a fresh runner, steps ran in order, and a failing step would have stopped the rest.

## Step 3 · Introduce a failure and watch the cascade

Add a deliberately failing step *after* the build:

```yaml
      - name: Deliberate failure
        run: exit 1
```

Commit + push. Open the new run:

- The `build` job turns **red**; the failing step is marked.
- Everything after the failure is **skipped** (this job has nothing after it — in Lab 03 you'll see how `needs:` propagates this to other jobs).

Now remove that step, commit + push, and confirm the run is **green** again.

## Step 4 · Run it manually

1. **Actions** tab → select the `CI` workflow → **Run workflow** (the *workflow_dispatch* trigger) → run on `main`.
2. Open the run: `event=workflow_dispatch` this time. Same workflow, different trigger — the `github.event_name` context changed.

## Step 5 · Prove the runner is ephemeral

Add this step and push:

```yaml
      - name: Wrote a file last run?
        run: |
          test -f /tmp/leftover && echo "LEFTOVER EXISTS" || echo "fresh runner — no leftover"
          touch /tmp/leftover
```

Run it twice. Both runs print **fresh runner — no leftover**: runners are wiped after every run. **Never treat a runner as persistent.**

## Step 6 · Map it to the model

Open `module-01-visualization.html` and run the **execution-model animation** (pick *push to main*). It replays exactly what you just watched in the UI. Then click blocks in the **Workflow file anatomy** explorer and confirm each block matches the file you wrote (`on`, `permissions`, `env`, `jobs`, `steps`).

---

## Expected outcome

- A green run on `ubuntu-latest` triggered by `push`, and a manual run via `workflow_dispatch`.
- You can point at any part of the workflow file and say which execution-model element it maps to.
- You have seen: event payload in `github.*`, failure stopping steps, and runner ephemerality.

## Key takeaways

- **Jobs** run on fresh runners; **steps** run in order; a failed step stops its job.
- The **`github` context** (event name, ref, sha) is injected into every run — you'll lean on it in every later lab.
- GitHub-hosted runners are **ephemeral and managed** — choose self-hosted only when you need private-network access or persistent state (guide §6).

## Troubleshooting

| Symptom | Fix |
|---|---|
| Run never appears after push | Check you committed the file under `.github/workflows/`; Actions only sees that path on the pushed branch |
| "The workflow is not valid" on the Actions tab | The YAML/schema validator in VS Code usually flags it first — fix there, then push |
| `git push` asks for credentials | Complete Setup 3 sign-in, or set a PAT/SSH key |
| Step fails on `npm init` (no network) | Corporate proxy must allow `registry.npmjs.org`; or replace the build step with `run: echo "build-ok"` for the lab |