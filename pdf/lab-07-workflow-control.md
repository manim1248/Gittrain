# Lab 07 · Workflow Control for Multi-Service Repos

> **Confidential · Stalwart Learning**
> Module 07 · Guided lab · Session 2
> Companion: `guides/module-07-workflow-control-multi-service-repos.md` · Visualization: `module-07-visualization.html`

| | |
|---|---|
| **Objective** | Turn `ci-demo` into a microservices monorepo and control CI with path filters, changed-file detection, a matrix build, `if:` gating, timeouts, and `concurrency` |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Lab 06 complete (you'll reuse `vars`/`secrets` habits) |
| **Files you create** | `services/checkout/*`, `services/orders/*`, `.github/workflows/checkout-ci.yml`, `.github/workflows/ci.yml` |

---

## Step 1 · Scaffold the monorepo

The Labs build two microservices under `services/`. Create the layout:

```bash
mkdir -p services/checkout services/orders
```

`services/checkout/package.json`:

```json
{
  "name": "checkout",
  "version": "0.1.0",
  "private": true,
  "scripts": { "test": "node -e \"console.log('checkout tests OK')\"" }
}
```

`services/orders/package.json`:

```json
{
  "name": "orders",
  "version": "0.1.0",
  "private": true,
  "scripts": { "test": "node -e \"console.log('orders tests OK')\"" }
}
```

Commit + push. Confirm a normal (non-filtered) workflow run is green.

## Step 2 · Option A — one workflow per service with path filters

Create `.github/workflows/checkout-ci.yml`:

```yaml
name: Checkout CI
on:
  push:
    branches: [main]
    paths:
      - 'services/checkout/**'
      - '!services/checkout/README.md'    # docs changes do NOT trigger CI
  pull_request:
    paths:
      - 'services/checkout/**'

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - run: npm test --prefix services/checkout
```

Now prove the filter:

1. Edit `services/checkout/package.json` (bump to 0.1.1), commit + push → **Checkout CI runs**.
2. Create `services/orders/README.md` with any content, commit + push → **no new run** — this is the **silent skip** of path filters (guide §2). Users often think CI is broken; it's "nothing matched".
3. Push a change under `services/checkout/README.md` → **still no run** (the `!` negation).
4. Push a change to `.github/workflows/checkout-ci.yml` itself → **it runs anyway** (workflow file changes always trigger).

> **ADO mapping:** `paths:` ≈ the *paths* filter on an ADO CI trigger, but Actions adds full glob control (`**`, `!`).

## Step 3 · Option B — one workflow, detect which service changed

Path filters decide *whether the workflow starts*. To gate jobs *inside* a workflow on *which* service changed, use a detection job. Create `.github/workflows/ci.yml`:

```yaml
name: Microservices CI
on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      checkout: ${{ steps.filter.outputs.checkout }}
      orders:   ${{ steps.filter.outputs.orders }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            checkout:
              - 'services/checkout/**'
            orders:
              - 'services/orders/**'

  build-services:
    needs: detect
    if: needs.detect.outputs.checkout == 'true' || needs.detect.outputs.orders == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix:
        include:
          - service: checkout
            dockerfile: services/checkout/Dockerfile
          - service: orders
            dockerfile: services/orders/Dockerfile
    steps:
      - uses: actions/checkout@v4
      - name: Build ${{ matrix.service }}
        run: |
          echo "building ${{ matrix.service }} from ${{ matrix.dockerfile }}"
          npm test --prefix services/${{ matrix.service }}

  notify-failure:
    needs: build-services
    if: always() && needs.build-services.result == 'failure'
    runs-on: ubuntu-latest
    steps:
      - run: echo "One or more services failed — notify the team"
```

Commit + push, then exercise it:

1. **Push touching only `services/orders/`** → `detect` runs (always), `build-services` runs and the run graph shows **only the `orders` matrix leg** (checkout was filtered out of the job).
2. **Push touching both** → both matrix legs run, in **parallel**, as distinct job instances (`build-services (checkout)`, `build-services (orders)`).
3. **Push touching neither** (e.g. a root README) → `build-services` is **skipped** but `detect` ran — this is the fix for the "required status check never posted" problem from the guide §5.

> **Why `dorny/paths-filter`?** It handles PR base-vs-head diffs and squashed pushes correctly — the raw `github.event.commits` approach is fragile (guide §3).

## Step 4 · Matrix controls — `fail-fast`, `include`, and a failure

1. In the visualization **7.2 Matrix builder**, toggle services and `fail-fast` to see the generated matrix and run branches you just used.
2. Make `orders` fail on purpose:

```yaml
      - name: Build ${{ matrix.service }}
        run: |
          echo "building ${{ matrix.service }} from ${{ matrix.dockerfile }}"
          npm test --prefix services/${{ matrix.service }}
          if [ "${{ matrix.service }}" = "orders" ]; then exit 1; fi
```

Push, open the run:

- With `fail-fast: false`, the **checkout leg is still green** — one service failing doesn't cancel the rest. You get every service's result in one run.
- `notify-failure` ran (orange) because of `if: always() && needs.build-services.result == 'failure'`.
- Revert the `exit 1` line and push.

> **ADO mapping:** `strategy.matrix` ≈ ADO **multi-configuration**; `include` ≈ per-combination extra values; `fail-fast: false` has no direct ADO equivalent — in ADO a failing leg cancels the job.

## Step 5 · Timeouts and the `if:` status functions

Add a timeout demonstration job to `ci.yml`:

```yaml
  slow:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - run: sleep 3600     # killed at 2 minutes
```

Push — `slow` is cancelled after 2 minutes ("exceeded the maximum execution time"). Remove it afterwards.

Then confirm the step-level cascade in `checkout-ci.yml`:

```yaml
      - run: exit 1                     # fails (temporarily)
      - run: echo "skipped by default"  # NOT run — previous step failed
      - run: echo "always runs"         # runs — explicit always()
        if: always()
```

Revert the `exit 1` before continuing.

> **Critical rule from the guide:** a step without `if:` only runs if all previous steps succeeded. ADO migrants expect "always run" — it's `if: always()` in Actions.

## Step 6 · `concurrency` — stop piling up runs

The `concurrency` block at the top of `ci.yml` already exists. Prove it:

1. Push **two commits within ~30 seconds** (e.g. `git commit --allow-empty` twice, then push both).
2. The **older run is cancelled** the moment the newer one starts — look for the grey *"This request was cancelled"* run.

This is the Actions equivalent of ADO's "cancel previous runs" and is essential on busy monorepos.

## Step 7 · Copilot checkpoint

> "Create a GitHub Actions workflow for a monorepo with services under `services/`. Add a `dorny/paths-filter` detection job with outputs per service, a matrix build job gated on those outputs with `fail-fast: false` and `timeout-minutes: 15`, and a failure-notification job using `needs` + `always()`."

Check the output (guide §10): are all service jobs gated on the detect job's outputs? Is `concurrency` present? Does the matrix `include` list every service with the right Dockerfile path? Copilot frequently forgets `concurrency` and `timeout-minutes`.

---

## Expected outcome

- A monorepo with `services/checkout` + `services/orders`.
- Both Option A (per-service path-filtered workflow) and Option B (detection + matrix) running; silent skips and partial-matrix runs observed and understood.
- `fail-fast`, `timeout-minutes`, `always()`/`failure()`, and `concurrency` all demonstrated with real runs.

## Key takeaways

- **Path filters decide whether a workflow starts; `dorny/paths-filter` decides which jobs run.**
- **`needs` builds the DAG** — jobs are parallel unless you wire dependencies; matrix legs are parallel job instances.
- **Always set `timeout-minutes`** and think about `fail-fast: false` for "build everything" workflows.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Workflow never starts after a push | Path filter didn't match — check the changed paths against the globs (guide §2), or `.github/workflows/**` changes always run |
| `build-services` skipped entirely | `detect` output false for both services, or the `if:` expression — check the `detect` job's outputs in the run |
| Matrix shows only one leg | `strategy.matrix` must be at **job** level; each `include` entry needs its `service`/`dockerfile` fields |
| `notify-failure` never runs on failure | It must be `needs: build-services` with `if: always() && needs.build-services.result == 'failure'` — a plain `needs:` job is skipped when the dependency fails |
| Old run not cancelled | `concurrency.group` must be identical across runs (here `ci-${{ github.ref }}`); `cancel-in-progress: true` must be set |