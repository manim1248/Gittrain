# Lab 08 · Artifacts & Caching

> **Confidential · Stalwart Learning**
> Module 08 · Guided lab · Session 2
> Companion: `guides/module-08-artifacts-and-caching.md` · Visualization: `module-08-visualization.html`

| | |
|---|---|
| **Objective** | Hand the built output from one job to the next with **artifacts**, and stop re-downloading npm dependencies with **caching** — and choose the right mechanism for each value you pass |
| **Time** | ~50 min (guided) |
| **Prerequisites** | Lab 07 complete (`services/checkout` with a committed `package-lock.json` — see Step 1) |
| **Files you create** | `.github/workflows/artifacts.yml`, `.github/workflows/cache-manual.yml` |

---

## Step 1 · Commit a lockfile (so caching has something to key on)

`actions/setup-node` with `cache: npm` keys the cache on `**/package-lock.json`. Generate and commit the lockfile for `checkout`:

```bash
npm install --prefix services/checkout     # creates services/checkout/package-lock.json
git add services/checkout
git commit -m "lab08: lockfile for checkout"
git push
```

Do the same for `services/orders` so both services are cache-ready.

## Step 2 · Artifacts — build hands off to test

Create `.github/workflows/artifacts.yml`:

```yaml
name: Artifacts & Caching
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
          cache-dependency-path: services/checkout/package-lock.json
      - name: Install
        run: npm ci --prefix services/checkout
      - name: Compute version
        id: version
        run: echo "version=v$(node -p "require('./services/checkout/package.json').version")" >> "$GITHUB_OUTPUT"
      - name: Build
        run: |
          mkdir -p services/checkout/dist
          echo "checkout build ${{ steps.version.outputs.version }}" > services/checkout/dist/out.txt
      - name: Upload dist as artifact
        uses: actions/upload-artifact@v4
        with:
          name: checkout-dist
          path: services/checkout/dist/
          retention-days: 14
          if-no-files-found: error

  test:
    needs: build
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
          cache-dependency-path: services/checkout/package-lock.json
      - run: npm ci --prefix services/checkout
      - uses: actions/download-artifact@v4
        with:
          name: checkout-dist
          path: services/checkout/dist/
      - name: Verify the handed-off build
        run: cat services/checkout/dist/out.txt
      - name: Version travelled via job output
        run: echo "deploying ${{ needs.build.outputs.version }}"
```

Commit + push, then open the run:

1. **Artifact hand-off:** `test` prints the exact `out.txt` that `build` produced on a *different runner*. Download it from the run page too — **Actions tab → run → Artifacts → `checkout-dist`** (a zip) — that's the "hand a build to a tester" nicety (guide §2).
2. **Job output vs artifact:** `deploying v0.1.0` came via `needs.build.outputs.version` — a *string* travelled without any artifact. Run the visualization **8.1 Artifact hand-off simulator** and confirm the shape.
3. Open `module-08-visualization.html` **8.2 Artifact vs output vs cache** and classify: `dist/` files → **artifact**; version string → **job output**.

> **ADO mapping:** upload/download-artifact ≈ `PublishPipelineArtifact` / `DownloadPipelineArtifact`; the run-page zip ≈ ADO's build "Artifacts" tab.

## Step 3 · Caching — watch the hit

1. Run the workflow a second time (push any trivial change, or use **Run workflow**).
2. Open the `build` job → expand **Install**:
   - First run: `npm ci` actually ran (cache miss — key was new).
   - Second run: `setup-node` logs `Cache restored from key: Linux-npm-<hash>` and `npm ci` is near-instant.
3. Check **Actions → Caches** (repo left nav): you'll see the entry with key `Linux-npm-<sha-of-lockfile>`.

> This is the built-in setup-action cache — one flag (`cache: npm` + `cache-dependency-path`) instead of the manual action (guide §3.2). The **monorepo trap** you just avoided: the `cache-dependency-path` points at *each service's* lockfile, not the repo root.

## Step 4 · Prove the key changes when dependencies change

1. Bump `services/checkout/package.json` to `0.2.0`, then locally run `npm install --prefix services/checkout` to regenerate the lockfile.
2. Commit + push. Open the run:
   - `setup-node` reports a **cache miss** — the `hashFiles(...)` ingredient changed, so the key changed.
   - `npm ci` re-downloads (slow run). Open visualization **8.3 Cache key simulator**: bump the lockfile version and watch the key change + miss.
3. Commit a revert back to `0.1.0` + lockfile; push.

> **Key design rule (guide §3.3):** a cache miss **must never break the build** — `npm ci` still works, just slower. The cache accelerates; it never defines correctness.

## Step 5 · Manual `actions/cache` with `restore-keys` and a `cache-hit` gate

The built-in cache is exact-key only. The manual action adds a **fallback**. Create `.github/workflows/cache-manual.yml`:

```yaml
name: Cache manual
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Cache ~/.npm
        id: npm-cache
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('services/checkout/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-
      - name: Install (skips on exact hit)
        run: npm ci --prefix services/checkout
      - name: Cache result
        run: echo "cache-hit=${{ steps.npm-cache.outputs.cache-hit }}"
```

Push and run twice:

1. Second run: `cache-hit=true` (exact key match).
2. Now change the lockfile (bump version, regenerate) and push: the key changes → `cache-hit=false`, but `restore-keys: ${{ runner.os }}-npm-` restores the **previous** cache as a partial fallback — only the diff re-downloads (guide §3.1 flowchart).

> **ADO mapping:** this is a near-1:1 map to ADO `Cache@2` (`key`, `path`, `restoreKeys`).

## Step 6 · Pitfalls to watch for (run the 8.4 matrix check)

- **Never cache the only copy of an output** — the cache is LRU-evicted and has no retention SLA; `dist/` went via *artifact*, not cache.
- **Matrix + artifacts:** if you upload the same artifact `name:` from every matrix leg, later uploads overwrite earlier ones. In Lab 09 the per-service names differ (`checkout-dist`, `orders-dist`) — or omit `name:` to download all.
- **Secrets in cache:** never cache `~/.docker/config.json` or `.npmrc` with tokens — the cache is not encrypted (guide §3.3).

## Step 7 · Copilot checkpoint

> "Add `actions/setup-node@v4` with `cache: npm` and `cache-dependency-path` pointing at `services/checkout/package-lock.json`. Add an `actions/cache@v4` step for `~/.npm` in the build job with a `hashFiles` key and a `restore-keys` fallback, gating `npm ci` on `cache-hit`."

Review (guide §7): is the key prefixed with `${{ runner.os }}`? Does it include the lockfile hash? Is there a `restore-keys`? Did it accidentally put the *output* in the cache instead of an artifact?

---

## Expected outcome

- `build` → `test` artifact hand-off, plus the version string via job output.
- A cache entry visible under Actions → Caches; a second-run hit observed.
- A deliberate lockfile change producing a cache miss; `restore-keys` fallback demonstrated.

## Key takeaways

- **Outputs → artifacts; dependencies → cache; strings → job outputs.** Choose by what you're handing over (guide §4).
- **Cache keys must change with dependencies** (`hashFiles(lockfile)`); a miss must never fail the build.
- Artifacts are downloadable from the run page and have `retention-days`; the cache is shared, unencrypted, and LRU-evicted.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `npm ci` fails: "lockfile not found" | Generate + commit `services/checkout/package-lock.json` first (Step 1) |
| Cache never hits | `cache-dependency-path` must point at the **service's** lockfile in a monorepo; the key only matches when the lockfile hash is identical |
| `cache-hit` always `false` | You're looking at the manual action's step output — set an `id:` on the `actions/cache` step and reference `steps.<id>.outputs.cache-hit` |
| Artifact overwritten by a matrix leg | Unique `name:` per combination (`${{ matrix.service }}-dist`), or download-all by omitting `name:` |
| Download path empty in `test` | `download-artifact` `path:` must be the same directory the upload consumed; artifacts download as a zip and unzip into `path` |
| `retention-days` not applied | `retention-days` is per-upload; the run page shows the effective expiry date |