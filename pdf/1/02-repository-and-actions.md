# Setup 2 · Repository and GitHub Actions

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration

## Objective

Create the repository every lab runs in, and confirm GitHub Actions is enabled and billing/runner defaults are sane.

## 1. Create the repository

1. Click **+ → New repository** (or **New** on the org page).
2. Owner: your personal account (or the org from Setup 1).
3. Name: `ci-demo` (all labs in this course assume this name; adjust in commands below if you differ).
4. **Public or Private**: both work. Private is fine and slightly closer to how you'll run Actions in production.
5. **Initialize with a README** and add a `.gitignore` for **Node** (the Module 02–03 labs build a Node service).
6. Click **Create repository**.

## 2. Confirm GitHub Actions is enabled

1. Open the **Actions** tab.
2. A page appears offering starter workflows ("Get started with GitHub Actions") — Actions is enabled.
3. If instead you see "Actions is not available" or a disabled workflow list, enable it:
   - **Settings → Actions → General → Actions permissions** → **Allow all actions and reusable workflows** (personal/org).
   - If your org restricts actions, use **Allow select actions** and add `actions/*`, `docker/*`, `rhysd/*`, `github/*` (or configure the org policy after the course).

> This default is deliberately permissive for a training sandbox. In production you'd pin actions and restrict to verified publishers (Session 3, Module 15) — the labs in Module 04 explore exactly that.

## 3. Check runner defaults (cost awareness)

- **Settings → Actions → General**:
  - *Default workflows permissions*: GitHub sets a **broad default** `GITHUB_TOKEN` permission set. Leave it for now — the Module 03/04 labs replace it with explicit least-privilege `permissions:` blocks.
  - *Fork pull request workflows*: `pull_request` (non-secret) is fine for the course.
- **Settings → Billing and plans** (or org **Billing**): note your included **Actions minutes** (private repos consume minutes; public repos are free).
  - Linux hosted runner minutes are the default pool; macOS/Windows cost 10x/2x. All labs use `ubuntu-latest`.

## 4. Clone locally

```bash
git clone https://github.com/<you-or-org>/ci-demo.git
cd ci-demo
git branch -m main      # if your default is not main
```

> If you used SSH: `git clone git@github.com:<you-or-org>/ci-demo.git`.

## 5. Smoke-test with your first workflow

This is the same workflow you will build properly in **Lab 01**. Create `.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, workflow_dispatch]
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Hello from GitHub Actions on ${{ runner.os }}"
```

Commit, push, then watch **Actions → CI** turn green.

```bash
git add .github/workflows/ci.yml
git commit -m "ci: first workflow"
git push
```

## Outcome

- `ci-demo` repo, cloned locally.
- First Actions run completed green on `ubuntu-latest`.
- You know where the Actions settings live and what the runner defaults are.

## Troubleshooting

| Problem | Fix |
|---|---|
| Actions tab shows starter list only | That's normal — your run appears once you push a workflow |
| Run never starts / "The workflow is not valid" | Validate YAML: the VS Code GitHub Actions extension (Setup 3) red-squiggles errors |
| `git push` fails authentication | Sign in to GitHub from the terminal (Setup 3) or use a personal access token / SSH key |