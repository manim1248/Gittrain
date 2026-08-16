# Lab 03 · Jobs, Steps & Execution Control

> **Confidential · Stalwart Learning**
> Module 03 · Guided lab · Session 1
> Companion: `guides/module-03-jobs-steps-execution-control.md` · Visualization: `module-03-visualization.html`

| | |
|---|---|
| **Objective** | Build a multi-job pipeline with `needs`, a matrix, artifacts, `$GITHUB_OUTPUT`, `if:` status functions, `timeout-minutes`, and `concurrency` — and watch the failure cascade |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Lab 01–02 complete |
| **Files you create** | `.github/workflows/pipeline.yml` |

---

## Step 1 · The DAG: parallel jobs + `needs`

Create `.github/workflows/pipeline.yml`:

```yaml
name: Pipeline
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "linting… OK"        # stand-in for a real linter (out of scope)

  test:
    needs: lint                        # waits for lint
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node: ['18', '20', '22']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: echo "tests on node ${{ matrix.node }} … OK"

  build:
    needs: lint                        # independent of test → runs parallel with it
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "building… OK" > build.txt
      - name: Publish build output
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: build.txt

  deploy:
    needs: [test, build]               # waits for both (and transitively lint)
    runs-on: ubuntu-latest
    environment: demo
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-artifact
      - name: Deploy
        run: cat build.txt
```

Commit + push. On the run page, look at the **job graph** (left panel):
- `lint` runs first; `test` (3 matrix jobs) and `build` run **in parallel**; `deploy` waits for all of them.
- Open `module-03-visualization.html` and run the **DAG simulator** — it models this exact shape.

> **Why `environment: demo`?** Environments bring approval gates (Session 3, Module 13). For now `demo` just tags the job as a deployment — it will show a "demo" environment in the run's Deployments list.

## Step 2 · Pass data: `$GITHUB_OUTPUT` between steps/jobs

Wire a value from `build` into `deploy` via a **job output**:

```yaml
  build:
    needs: lint
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.meta.outputs.version }}   # exposed to downstream jobs
    steps:
      - run: echo "version=v1.0.0" >> "$GITHUB_OUTPUT"
        id: meta
```

and in `deploy`:

```yaml
      - name: Show inherited output
        run: echo "deploying version ${{ needs.build.outputs.version }}"
```

Push and confirm `deploy` prints `v1.0.0`. Compare: steps talk via `$GITHUB_OUTPUT`, jobs talk via **job outputs** + `needs`, files travel via **artifacts** (already doing that).

## Step 3 · Conditional execution: `if:` + status functions

Extend `deploy` with three conditional steps:

```yaml
      - name: Only on main
        if: github.ref == 'refs/heads/main'
        run: echo "production path on main"
      - name: Notify on failure
        if: failure()
        run: echo "FAILED — page the team (stand-in for a Slack/PagerDuty action)"
      - name: Always-run cleanup
        if: always()
        run: echo "cleanup (guaranteed to run)"
```

Push. On the green run, check each step's **status badge**: the `failure()` step shows **skipped**, the `always()` step ran, `Only on main` ran.

## Step 4 · Make it fail — observe the cascade

Temporarily break the `test` job:

```yaml
      - run: exit 1
        if: matrix.node == '20'    # only the Node 20 leg fails
```

Push and open the run:
- The **Node 20** test leg is red; Node 18 and 22 are still green (`fail-fast: false`).
- `deploy` shows **skipped** — `needs: [test, build]` means a failed dependency blocks it (the `failure()` step never runs because the whole job is skipped).
- Revert the `exit 1` step and push.

## Step 5 · `timeout-minutes` and `continue-on-error`

```yaml
  slow:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - run: sleep 3600          # will be killed at 2 minutes
```

Push. The `slow` job is **cancelled by the timeout** after 2 minutes (you'll see "The job running on runner … has exceeded the maximum execution time").

Then add a non-blocking check with `continue-on-error`:

```yaml
  vuln-scan:
    runs-on: ubuntu-latest
    continue-on-error: true     # failing scan does NOT fail the run
    steps:
      - run: echo "scan failing…" && exit 1
```

Push — the run is **green overall** even though the scan "failed".

## Step 6 · `concurrency`: cancel the older run

Add at the top of `pipeline.yml`:

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

Now push **twice in quick succession** (two commits within ~30s). The **older run is cancelled** the moment the newer one starts — look for the grey "This request was cancelled" run. This is the standard guard for CI on a busy branch.

---

## Expected outcome

- A 5-job DAG (lint → test×3 + build → deploy) with artifact passing, job outputs, conditionals, a timeout kill, a non-blocking scan, and concurrency cancellation.
- You have *caused* a failure and watched `needs`-based skipping, and you can predict each status function's outcome from the module-03 `if:` explorer table.

## Key takeaways

- **Jobs are the unit of parallelism**; `needs` builds the DAG. Job graph ≠ stages — use environments for deployment targets.
- **Three data channels:** steps ↔ `$GITHUB_OUTPUT`, jobs ↔ outputs + `needs`, files ↔ artifacts (or `actions/cache` for deps, Session 2).
- `always()`/`failure()`/`cancelled()` + `continue-on-error` give you fine-grained failure handling; `timeout-minutes` prevents runaway jobs; `concurrency` prevents pile-ups.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `needs.build.outputs.version` empty | The producing step must have an `id:` AND the job must declare `outputs:` — both are required |
| Deploy skipped but you wanted it to run | Default `if:` is `success()`; add `if: always()` (or `failure()`) deliberately, or fix the failing dependency |
| Matrix shows only one leg | `strategy.matrix` must be at job level with the value referenced as `${{ matrix.<key> }}` |
| `slow` never times out | `timeout-minutes` is per **job** (not step); also check the Actions setting *Job timeout* default isn't overridden lower |