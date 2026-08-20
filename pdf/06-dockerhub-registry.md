# Setup 6 · Docker Hub Registry — Setup and GitHub Integration

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration

## Objective

Create the Docker Hub identity, repositories and **access token** the Session 2 workflows push to, wire them into GitHub as secrets, and prove the login→push loop works from your terminal **before** Lab 09 automates it.

> **Why Docker Hub?** It needs no Azure subscription (ACR + OIDC is Session 3) and no extra permissions (unlike GHCR's `packages: write`), so it is the frictionless registry for the container labs. The Module 09 guide shows the GHCR/ACR alternatives — the Docker Hub pattern below is identical apart from the `username`/`password` values.

## 1. Docker Hub account

1. Go to **https://hub.docker.com** → **Sign Up** (or sign in with your team's account).
2. Your **username is the namespace**: image references look like `<your-dockerhub-username>/checkout-service`.
3. The **Free** plan is enough for the labs. Free gives one **private** repo — make the lab repos **public**, or upgrade if you need private images.

> Docker Hub has pull **rate limits** on free plans (100 pulls/6 h per IP). The labs only push and pull a few small images — stay under the limit by deleting old test tags, or use a paid plan / your team's org if you hit "toomanyrequests".

## 2. Create an access token (never use your password)

1. hub.docker.com → **Account Settings** (avatar, top-right) → **Security** → **New Access Token**.
2. Description: `github-actions-ci`. Access scope: **Read, Write, Delete** (lab 09 pushes; Write is the minimum).
3. Click **Generate** → **copy the token immediately** — it is shown **once**.
4. Store it in your password manager.

> **Why a token and not the password?** Tokens are revocable and scoped — you can rotate one without changing your account password. Your Docker Hub **password must never** be added to GitHub secrets or pasted into a workflow. (Also note: `docker login` on a Docker-Hub token does **not** trigger 2FA prompts, which is exactly why CI uses it.)

## 3. Create the repositories

1. hub.docker.com → **Repositories** → **Create repository** (top-right).
2. Create **two** public repositories: `checkout-service` and `orders-service`.
3. Leave **"Automated builds"** off — CI pushes from GitHub Actions, not from Docker Hub builds.

> Push **does not auto-create** repositories on Docker Hub (unless your account has auto-create enabled). Create them in the UI first; otherwise `docker push` fails with *"denied: requested access to the resource is denied"*.

## 4. Verify credentials locally (integration check)

This proves the token + repos work **before** you automate them:

```bash
docker login -u <your-dockerhub-username>
# password prompt → paste the ACCESS TOKEN (not your password)

docker tag hello-world <your-dockerhub-username>/checkout-service:local-check
docker push <your-dockerhub-username>/checkout-service:local-check
docker pull <your-dockerhub-username>/checkout-service:local-check
```

Check hub.docker.com → `checkout-service` → Tags — you should see `local-check`.

## 5. Wire into GitHub — the integration

The workflows read two **repository secrets**. Add them to `ci-demo`:

**Repo `ci-demo` → Settings → Secrets and variables → Actions → New repository secret:**

| Secret name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | your Docker Hub username (e.g. `yourcompany-devops`) |
| `DOCKERHUB_TOKEN` | the access token from step 2 |

> The username isn't really sensitive — you *could* put it in a variable — but keeping both together as secrets is simpler and is how the labs reference them. Org-level secrets (shared across repos) come in Lab 06.

## 6. How workflows consume these (preview — full build in Lab 09)

```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    push: true
    tags: ${{ secrets.DOCKERHUB_USERNAME }}/checkout-service:sha-${{ github.sha }}
```

Rules that follow from the Module 06 guide:

- Reference secrets only inside `${{ }}` — never `echo` them (masking kicks in **after** first reference; a value printed earlier can leak).
- An **unset** secret resolves to an **empty string** — a blank `DOCKERHUB_TOKEN` produces a silent `unauthorized` in CI. Guard it (Lab 06 Step 4) and verify it's set before running Lab 09.
- Secrets are masked as `***` in logs **after** their first reference — the token will appear masked if an auth failure is printed.

## Outcome / verification checklist

- [ ] `docker login` succeeds with the **access token**
- [ ] `docker push`/`docker pull` of `checkout-service:local-check` succeed and the tag shows on hub.docker.com
- [ ] GitHub secrets `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` exist on `ci-demo`
- [ ] `docker run --rm hello-world` still works (Setup 5)

You are ready for Lab 06 (secret masking, scopes) → Lab 09 (build & push) → Lab 10 (shared workflows).

## Troubleshooting

| Problem | Fix |
|---|---|
| `docker push` → *"denied: requested access to the resource is denied"* | The repository does not exist — create it in the UI (§3), or the token lacks **Write** scope |
| `docker login` → *"unauthorized: incorrect username or password"* | You pasted your account **password**, not the token; or the token is truncated — re-generate (§2) |
| `docker login` keeps asking for 2FA | You're using the password — use the token instead; tokens bypass 2FA prompts |
| Pull hits *"toomanyrequests"* | Free-plan rate limit — wait, delete old tags, or use a paid/org account |
| CI push fails but local push works | `DOCKERHUB_TOKEN` empty or wrong in GitHub — re-check Settings → Secrets, and guard with the Lab 06 check |
| Tag not visible in hub.docker.com after CI push | Refresh the Tags tab; verify the workflow ran with `push: true` (Lab 09) and not only on `pull_request` |
