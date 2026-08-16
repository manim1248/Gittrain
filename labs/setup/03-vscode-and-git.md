# Setup 3 · VS Code, Git and GitHub Integration

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration

## Objective

Wire up the editor you will author workflows in: VS Code, Git, GitHub authentication, and the two extensions that make workflow authoring live (validation + Copilot).

## 1. Install prerequisites

- **VS Code** — https://code.visualstudio.com → download → install.
- **Git client** (latest stable) — https://git-scm.com. Verify: `git --version`.
- **Docker Desktop** — https://www.docker.com/products/docker-desktop. Needed for Session 2 container builds; install now so the lab environment is complete. Verify: `docker --version`.

## 2. Install the extensions

Open VS Code → **Extensions** (⇧⌘X / Ctrl+Shift+X) → install:

| Extension | ID | Why |
|---|---|---|
| **GitHub Actions** | `GitHub.vscode-github-actions` | Workflow **syntax + schema validation**, run explorer, workflow picker |
| **YAML** | `redhat.vscode-yaml` | YAML validation + formatting (feeds the Actions extension) |
| **GitHub Pull Requests and Issues** | `GitHub.vscode-pull-request-github` | Open PRs from the editor, required for the Session 2 PR-flow labs |
| **GitHub Copilot** | `GitHub.copilot` | Inline completions (Module 05) |
| **GitHub Copilot Chat** | `GitHub.copilot-chat` | The Chat panel you'll drive the Module 05 lab with |

> The Copilot extensions will show a sign-in/status warning until Setup 4 — that is expected.

## 3. Sign in to GitHub from VS Code

1. Click the **Accounts** icon (bottom-left) → **Sign in with GitHub**.
2. Authorize in the browser. This gives VS Code push/pull, PR, and Actions access.
3. Verify: the accounts menu shows your avatar, and `git config --global user.name` / `user.email` are set (or set them now):
   ```bash
   git config --global user.name "Your Name"
   git config --global user.email "you@example.com"
   ```

## 4. Open the repo and confirm the Actions integration

1. **File → Open Folder →** the `ci-demo` folder from Setup 2.
2. Open `.github/workflows/ci.yml`:
   - You should see YAML syntax highlighting.
   - A green/yellow **Diagnostics** underline appears if the schema validator flags a problem (try typing `ron: [push]` to see it react).
3. Open the **GitHub Actions** extension icon in the Activity Bar — it lists your workflow files and recent runs.

## 5. Verify live run awareness

- Make a trivial edit to `ci.yml`, commit + push from VS Code's Source Control panel.
- The GitHub Actions extension shows the new run; opening it links to the run page.

## Outcome

- VS Code with Git, Actions, YAML, PR, Copilot + Copilot Chat extensions.
- GitHub authenticated in the editor.
- Schema validation working on `.github/workflows/ci.yml`.

## Troubleshooting

| Problem | Fix |
|---|---|
| No schema validation | Ensure both **GitHub Actions** and **YAML** extensions are enabled; reopen the file; `Cmd+Shift+P` → "GitHub Actions: …" should appear |
| Sign-in loops | Check the browser isn't blocking the `vscode://` callback; re-try from the Accounts menu |
| Docker not running | Start Docker Desktop; the Session 2 lab will confirm with `docker run hello-world` |