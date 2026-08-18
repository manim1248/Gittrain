# Lab Setup — GitHub Actions Environment Provisioning

> **Confidential · Stalwart Learning**
> GitHub Actions — CI/CD Enablement & Migration · **Setup for Session 1–2**

Complete these steps **before** the relevant session. Sessions are instructor-led and demo-driven — you follow along on your own account where time permits.

- **Steps 1–4** → before **Session 1** (Modules 01–05).
- **Steps 5–6** → before **Session 2** (Modules 06–10, Docker + Docker Hub container builds).

## What you will set up

| # | Step | Doc | Why you need it |
|---|------|-----|-----------------|
| 1 | GitHub account (personal + optional org) | `01-github-account.md` | Everything runs on github.com |
| 2 | Repository + GitHub Actions enabled | `02-repository-and-actions.md` | Where your workflows live and run |
| 3 | VS Code + Git + GitHub integration | `03-vscode-and-git.md` | The editor you author workflows in |
| 4 | GitHub Copilot (Business/Enterprise) | `04-github-copilot.md` | Module 05 uses Copilot from day one |
| 5 | Docker Desktop (local verification + integration) | `05-docker-desktop.md` | Session 2 container builds (Modules 08–10) |
| 6 | Docker Hub registry + GitHub integration | `06-dockerhub-registry.md` | Push images in Labs 09–10 |

## Prerequisites (from the course outline)

- A browser with unrestricted access to `github.com` (arrange corporate proxy/firewall exceptions **before** Session 1).
- **Git client** (latest stable).
- **Docker Desktop** + a **Docker Hub account** — for the container build walkthroughs in Session 2 (steps 5–6; not needed for Session 1). Allow-list `hub.docker.com`, `registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com` if your org proxies registry traffic.
- An **Azure subscription + container registry** (existing sandbox/dev subscription is fine) — needed only from Session 3 (OIDC).
- A GitHub account with **permissions to create repositories and enable Actions** in at least one org/repo.
- **GitHub Copilot licence (Business or Enterprise)** enabled for the account being used.

> If your org has not yet purchased Copilot for GitHub, ask your org admin to start a **Copilot Business 30-day trial** for the participating accounts — that covers Module 05 and the AI-assisted authoring labs. An individual **Copilot Free** licence is a fallback for the guided labs, but some org-level features (Module 04 reusable workflows with org secrets) behave differently.

## Time

~45–60 minutes total for steps 1–4 (one time, before Session 1). Steps 5–6 add ~30 minutes before Session 2. Steps 3–4 can run in parallel with the instructor.

## Where the labs are

After setup, each module has a guided lab in `labs/module-0N/`:

- `module-01/lab-01-architecture.md`
- `module-02/lab-02-events-triggers.md`
- `module-03/lab-03-jobs-control.md`
- `module-04/lab-04-actions-reuse.md`
- `module-05/lab-05-copilot.md`
- `module-06/lab-06-variables-secrets-contexts.md`
- `module-07/lab-07-workflow-control.md`
- `module-08/lab-08-artifacts-caching.md`
- `module-09/lab-09-container-build-push.md`
- `module-10/lab-10-reusable-workflows.md`

Each lab references its companion guide (`guides/module-0N-*.md`) and interactive visualization (`module-0N-visualization.html`). Labs 06–10 build a microservices monorepo (`services/checkout`, `services/orders`) and push images to **Docker Hub** using the credentials from Setup 6.

## Quick verification checklist (do once, after all setup docs)

- [ ] Can sign in to `github.com`
- [ ] Created a repo and cloned it locally
- [ ] `git push` from VS Code triggers a **new Actions run** you can watch
- [ ] VS Code shows the GitHub Actions extension icon in the Activity Bar
- [ ] Copilot Chat answers a prompt (e.g. *"explain what actions/checkout does"*)
- [ ] `docker run --rm hello-world` succeeds (Setup 5)
- [ ] `docker login -u <you>` works with the **access token**; `docker push` of a test tag shows on hub.docker.com (Setup 6)
- [ ] Repo secrets `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN` are set on `ci-demo` (Setup 6)