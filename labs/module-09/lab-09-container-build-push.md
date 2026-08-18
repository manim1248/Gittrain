# Lab 09 · Container Build & Push (Docker Hub)

> **Confidential · Stalwart Learning**
> Module 09 · Guided lab · Session 2
> Companion: `guides/module-09-container-build-pipelines.md` · Visualization: `module-09-visualization.html`

| | |
|---|---|
| **Objective** | Build a Docker image for `services/checkout`, tag it with an immutable strategy, and push it to **Docker Hub** — from your terminal first, then automated in a workflow with Buildx, login, metadata and the GitOps hand-off boundary |
| **Time** | ~60 min (guided) |
| **Prerequisites** | Labs 06–08 complete · **Setup 5 (Docker Desktop) + Setup 6 (Docker Hub)** verified |
| **Files you create** | `services/checkout/app.js`, `services/checkout/Dockerfile`, `.github/workflows/build-push.yml` |

---

## Step 1 · A real service + Dockerfile, proven locally

Create `services/checkout/app.js`:

```js
const http = require('http');
http.createServer((req, res) => res.end('checkout-service OK\n'))
  .listen(process.env.PORT || 8080);
```

Create `services/checkout/Dockerfile`:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY app.js ./
EXPOSE 8080
CMD ["node", "app.js"]
```

> Notice the pattern you'll keep in CI: `COPY package*.json` first, `npm ci`, *then* copy the code — so dependency-layer changes are rare and Buildx layer caching (Step 4) actually helps.

Now **prove it locally** — this is the Docker integration check from Setup 5/6:

```bash
docker build -t <your-dockerhub-username>/checkout-service:local-test \
  -f services/checkout/Dockerfile services/checkout

docker run --rm -p 8080:8080 <your-dockerhub-username>/checkout-service:local-test &
curl -s localhost:8080        # → checkout-service OK
kill %1

docker push <your-dockerhub-username>/checkout-service:local-test
```

If the `push` fails, your repo or token isn't right — fix it **before** the workflow (Setup 6 §4/§7).

> **Architecture note:** on Apple silicon your local image is `linux/arm64`. CI (with Buildx, Step 4) builds `linux/amd64` too, which is what the AKS cluster consumes. Don't be surprised that a local image doesn't match the pushed tags — that's the multi-platform default of `build-push-action`.

## Step 2 · The workflow — build, tag, push, hand off

Create `.github/workflows/build-push.yml`:

```yaml
name: Checkout — Build & Push
on:
  push:
    branches: [main]
    paths: ['services/checkout/**']
  workflow_dispatch:

permissions:
  contents: read

env:
  IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/checkout-service

jobs:
  build-push:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=ref,event=branch
            type=sha,format=short

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: services/checkout
          file: services/checkout/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  gitops-handoff:
    needs: build-push
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Flux hand-off — bump the image ref in the manifest repo to:"
          echo "${{ env.IMAGE }}:sha-${{ github.sha }}"
```

Commit + push the workflow **and** the `services/checkout` files. Use **Run workflow** (dispatch) so the run isn't blocked by the path filter — then watch:

1. **setup-buildx** installs BuildKit.
2. **Login to Docker Hub** succeeds — the token is masked in the log (only `***` if printed).
3. **Extract metadata** computes tags for this run (branch `main` + short sha).
4. **Build and push** pushes `:main` and `:sha-<7>` to Docker Hub. Confirm both on hub.docker.com → `checkout-service` → **Tags**.
5. **gitops-handoff** prints the immutable ref `your-user/checkout-service:sha-<sha>` — this is the **Actions → Flux boundary** (guide §7): Actions updates the manifest *reference*; Flux reconciles AKS. No `kubectl apply` in CI.

> **ADO mapping:** `build-push-action` ≈ `Docker@2`; Docker Hub login with secrets ≈ an ACR/Docker Hub **service connection**; `github.sha`/`github.run_number` ≈ `$(Build.SourceVersion)`/`$(Build.BuildNumber)` tags.

## Step 3 · Tagging strategy — why `sha` and not `latest`

Open `module-09-visualization.html` **9.2 Tagging strategy** and toggle the strategies; then compare with what you pushed:

| Tag you got | Meaning | Why you keep it |
|---|---|---|
| `main` | branch pointer | "what's latest on main" — convenient, mutable |
| `sha-<7>` | commit pointer | **immutable traceability** — Flux diffs on this ref |
| (semver `vX.Y.Z`) | release | on a tag push — not shown here (guide §4) |
| `latest` | — | **avoid as a deploy target** — non-immutable, breaks rollbacks |

Add the semver rule to the metadata block and push a tag to see it:

```yaml
          tags: |
            type=ref,event=branch
            type=sha,format=short
            type=semver,pattern={{version}}
```

```bash
git tag v0.1.0 && git push origin v0.1.0
```

The tag push triggers a run that now also pushes `v0.1.0` to Docker Hub.

> **Why it matters for GitOps:** Flux watches your manifest repo for a *changed image reference*. `sha` makes every push a unique immutable reference; `latest` makes the hand-off ambiguous and rollbacks impossible (guide §4).

## Step 4 · Prove the layer cache

Run the workflow again (empty commit + push, or dispatch):

- In the **Build and push** step, layers unchanged since the last run show as `CACHED` (Buildx + `cache-from/to: type=gha`).
- The `gha` cache lives in the GitHub Actions cache (Module 08) — run `docker buildx history`/the step log to see "exporting to cache" and "CACHED".

Without this cache every run would rebuild every layer (guide §8 pitfall 3).

## Step 5 · Gate the push — never push on PRs

The workflow currently triggers on `push` to `main` + dispatch only, so `push: true` is safe. For the general case, gate the *push* (guide §8 pitfall 6):

```yaml
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
          # build still happens on PRs; only main pushes to the registry
```

Leave the trigger as-is for this lab — but note the pattern for Lab 10's shared workflow.

## Step 6 · Copilot checkpoint

> "Create a Docker build-and-push workflow for `services/checkout` using `docker/setup-buildx-action@v3`, `docker/login-action@v3` with `${{ secrets.DOCKERHUB_USERNAME }}` / `${{ secrets.DOCKERHUB_TOKEN }}`, `docker/metadata-action@v5` with branch + sha tags, and `docker/build-push-action@v6` with `type=gha` cache. Gate the push to `push` on `main`."

Review the output (guide §9): is `permissions` set? Is the build `context` the **service directory**, not the repo root? Does the tag set include a `sha` tag? Is the cache block present?

---

## Expected outcome

- The same image you built locally is pushed by CI to Docker Hub with `main` + `sha-<7>` (+ `v0.1.0` after the tag).
- A run showing Buildx `CACHED` layers on the second build.
- The GitOps hand-off ref computed and printed — the Actions→Flux boundary.

## Key takeaways

- **`docker/login-action` + secrets** is the secret-based registry auth (Docker Hub); GHCR uses `GITHUB_TOKEN` + `packages: write`, ACR uses OIDC (Session 3) — same workflow shape, different `with:`.
- **Tag by `sha` (immutable) + semver; never deploy from `latest`.**
- **Buildx + `type=gha` cache** makes rebuilds fast; `context:` must point at the service dir.
- Actions ends at the **manifest-ref hand-off** — Flux does the deployment.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `denied: requested access to the resource is denied` | Repo doesn't exist on Docker Hub, or token lacks **Write** — create it / re-scope (Setup 6) |
| `unauthorized: incorrect username or password` | `DOCKERHUB_TOKEN` empty or wrong — check Settings → Secrets, run the Lab 06 guard step |
| Login works locally but not in CI | Local `docker login` used a different stored credential; ensure the **secrets**, not your password, are referenced in `with:` |
| `CACHED` never appears | First run on a fresh cache always rebuilds; confirm `cache-from: type=gha` and `cache-to: type=gha,mode=max` are both set |
| Image runs locally but fails on AKS | Architecture mismatch — local `linux/arm64` vs cluster `linux/amd64`; rely on CI's multi-platform Buildx build, not the local one |
| Push happens on PR runs | You removed the `paths`/`if:` gating — re-add Step 5's `push:` expression or keep the main-only trigger |