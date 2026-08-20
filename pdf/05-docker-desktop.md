# Setup 5 · Docker Desktop — Setup and Integration

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration

## Objective

Install and configure Docker locally so you can (a) reproduce the container-build pipeline on your own machine and (b) prove registry credentials work **before** the workflows in Lab 09/10 push images. Docker Desktop gives you the same `docker` CLI + BuildKit that the `ubuntu-latest` GitHub-hosted runner uses — so a command that works locally will work in Actions.

> Where this fits: the GitHub-hosted runner already has Docker and Buildx pre-installed — this local setup is **not** for running Actions, it is your verification sandbox for the Session 2 container labs (Modules 08–10) and for Docker Hub integration (Setup 6).

## 1. Install Docker Desktop

- **macOS** (Apple silicon or Intel): download from https://www.docker.com/products/docker-desktop → open the `.dmg` → drag **Docker** into **Applications** → launch from Launchpad.
- **Windows**: install **WSL2 first** (run `wsl --install` in an elevated PowerShell and reboot), then Docker Desktop (choose the **WSL2 backend** when prompted).
- **Linux**: Docker Engine for your distro (`docker-ce` or `docker.io`) plus the **buildx** plugin; Docker Desktop is optional on Linux.

Verify all three components:

```bash
docker --version        # Docker version 2x.y.z
docker buildx version   # buildx v0.x — required by Lab 09/10
docker compose version  # v2 — optional, not used by the labs
```

## 2. Start and configure

1. Launch Docker Desktop and accept the licence. Wait until the whale stops animating and the status reads **Docker Desktop is running**.
2. **Settings → Resources**: give it **RAM ≥ 8 GB** and **4+ CPUs** (4 GB / 2 CPUs minimum). The container labs are small, but low resources make `npm ci` inside images feel sluggish.
3. **Settings → Builds**: leave **BuildKit** enabled (default). Confirm a default builder exists:

```bash
docker buildx ls        # should list a 'default' builder
```

4. Keep Docker Desktop **running during the Session 2 labs** when you need local verification (Lab 09 Step 1). The workflows themselves run on GitHub's runners and don't touch your local daemon.

## 3. Smoke-test

```bash
docker run --rm hello-world
```

- On success you see "Hello from Docker!" and "This message shows that your installation appears to be working correctly".
- Then confirm Buildx can build (this also downloads the buildx docker driver if missing):

```bash
docker buildx build --help >/dev/null && echo "buildx OK"
```

## 4. Integration with the GitHub Actions labs

How this local install plugs into the course:

| Local Docker is used for | The GitHub runner is used for |
|---|---|
| Lab 09 Step 1: build + run + push the service image manually | Running the workflows themselves (buildx, login, push) |
| Verifying Docker Hub credentials (Setup 6 §4) before automation | The `docker/*` actions (`setup-buildx`, `login`, `metadata`, `build-push`) |
| Debugging a Dockerfile that fails in CI | Layer caching via `cache-from/to: type=gha` |

- **`type=gha` cache is not shared with your laptop.** Labs 08–10 use the GitHub Actions cache for image layers; your local `docker build` won't reuse it, and that's expected.
- **Registry allow-list for corporate proxies:** the runner and your laptop both need `registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com`, `hub.docker.com` (and `*.githubusercontent.com`, `*.actions.githubusercontent.com` for Actions itself). Ask your network team **before** Session 2.
- **Architecture note:** `ubuntu-latest` runners are `linux/amd64`. If you are on Apple silicon, images you build locally are `linux/arm64` — Lab 09's `docker/build-push-action` with Buildx builds **both** (`linux/amd64` + `linux/arm64`) unless you set `platforms:`, so the pushed image matches the AKS cluster. Keep this in mind if a workflow-built image runs on your Mac but fails on the cluster.

## Outcome

- `docker --version`, `docker buildx version`, and `docker run --rm hello-world` all succeed.
- Docker Desktop stays running; you can run the exact `docker` commands the workflows will run (Lab 09 Step 1).
- You are ready for **Setup 6 · Docker Hub** (registry account, token, repositories, GitHub secrets).

## Troubleshooting

| Problem | Fix |
|---|---|
| `docker` command not found | Restart the terminal / VS Code so the new PATH is picked up; on Windows use a WSL2 terminal |
| "Docker Desktop is starting…" forever | Quit and relaunch; on macOS check **Settings → Resources** isn't oversubscribed; on Windows confirm WSL2 (`wsl -l -v`) |
| `docker run hello-world` pulls but hangs | Check the registry allow-list (see §4); try `docker pull hello-world` with a proxy set |
| `docker buildx ls` empty | Run `docker buildx create --use` once, or update Docker Desktop |
| Port conflict starting containers | `docker ps -a`, remove stray containers, or change the published port in Lab 09 (`-p 8081:8080`) |
