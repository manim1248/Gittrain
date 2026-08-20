# Lab 16 · Monitoring & Debugging — Read It, Re-run It, Get Alerted

> **Confidential · Stalwart Learning**
> Module 16 · Guided lab · Session 4
> Companion: `guides/module-16-monitoring-and-debugging.md` · Visualization: `module-16-visualization.html`

| | |
|---|---|
| **Objective** | Operate a workflow the way you'll run it every day: read the run summary and job graph, drill into logs, re-run a failed matrix cell, turn on debug logging, and configure alerting. You'll deliberately break a workflow to learn the recovery motions — safely |
| **Time** | ~50 min (guided) |
| **Prerequisites** | Setups 02 (repo + Actions), 03 (VS Code + git). A green workflow from any earlier lab (Labs 09–15) to practice on. Copilot (Setup 04) for Step 7 |
| **Files you create** | `.github/workflows/monitoring-demo.yml` (the "breakable" workflow), then a fix commit; optionally a `workflow_run` notifier workflow |

---

## Step 1 · Anatomy of a run

Open visualization **16.1** and click each part of the run summary — header, job status, annotations, step logs. Then open one of your earlier green runs in the Actions tab and map what you see onto the four parts.

> **ADO mapping (guide §2):** ADO's *Runs → job logs* and timeline map onto Actions' run summary + job graph. The new muscle is the **PR checks widget** — your status checks (Module 14) are first-class on the pull request.

## Step 2 · Build the breakable workflow

Create `.github/workflows/monitoring-demo.yml` — a matrix job whose cells fail **intermittently** (so you learn recovery without any real damage):

```yaml
name: Monitoring demo
on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: [orders, payments, inventory]
    steps:
      - uses: actions/checkout@v4
      - name: Simulate build
        run: sleep 2 && echo "build ok (${{ matrix.service }})"
      - name: Simulate flaky test
        run: |
          sleep 1
          if [ $(( RANDOM % 3 )) -eq 0 ]; then
            echo "test failed for ${{ matrix.service }}"
            exit 1
          fi
          echo "test passed for ${{ matrix.service }}"
```

Push, then trigger the workflow twice via **Actions → Monitoring demo → Run workflow** (or `workflow_dispatch`). Keep the runs until at least one cell fails.

> **Why `fail-fast: false` matters (guide §4):** with it, a red cell doesn't cancel its siblings — you see *all* failures in one run instead of one at a time. That's the debugging-friendly default while you're learning.

## Step 3 · Read the job graph

Open the **failing run** (not the success one). Open visualization **16.2**, run the simulated failure, and click jobs to see what they block. Now do the same on your real run:

1. Scan the **job status summary** — which cells are red? Which are green (kept running)?
2. Confirm a **cancelled/grey** run is *not* a failure — did a human stop it or a timeout fire?
3. Note the **step timings** — the `Simulate flaky test` step's duration and exit code.

## Step 4 · Diagnose from the log tail

Open the failing cell → the `Simulate flaky test` step → read the **last few lines**:

```text
test failed for orders
Error: Process completed with exit code 1.
```

That's the whole diagnosis for this synthetic failure — a real failure will be noisier, but the motion is identical: **open the failing step, read the tail, find the exit code**.

## Step 5 · Re-run — granularity and matrix cells

Open visualization **16.3** and toggle the three re-run modes, then **16.4** and click cells. On your real run:

1. **Re-run failed jobs** (Actions → run → ⚙ → *Re-run failed jobs*). The green cells keep their result; only the failed cell replays. Watch the graph.
2. If a cell fails again, try **Re-run all jobs** — everything executes fresh.
3. Note that re-running from a **failed step** would replay only the failing step — useful while iterating on a fix.

> **ADO mapping (guide §3):** ADO re-runs the *whole stage*. Actions lets you re-run just the failed jobs — or one matrix cell. This is a daily quality-of-life win you'll feel immediately.

## Step 6 · Turn on debug logging, then off

For a real post-mortem you want full detail. In **Settings → Actions → General → Workflow permissions / logging**, enable *Runner diagnostic logging* and *Step debug logging*, re-run, and compare the log depth. Then:

1. **Download logs** (run → ⚙ → Download logs) and grep the zip locally: `zgrep -n "exit" *.log`.
2. **Turn debug logging back off** — verbose logs, slower runs, and a bigger secret-exposure surface (Module 15) when left on.

## Step 7 · Alerting — from in-product to webhook

Open visualization **16.6** and click each delivery layer. Configure at least two:

1. **In-product** — Settings → Notifications → repository → Actions: receive notifications on failed runs (zero setup).
2. **Webhook/notifier workflow** — create `.github/workflows/notify.yml` that reacts when another workflow completes:

```yaml
name: Notify on failure
on:
  workflow_run:
    workflows: ["Monitoring demo"]
    types: [completed]

permissions:
  actions: read

jobs:
  alert:
    runs-on: ubuntu-latest
    if: github.event.workflow_run.conclusion == 'failure'
    steps:
      - name: Print failure card
        run: |
          echo "🚨 Monitoring demo FAILED: ${{ github.event.workflow_run.html_url }}"
      - name: Post to Teams (optional)
        if: secrets.TEAMS_WEBHOOK != ''
        run: |
          curl -X POST "${{ secrets.TEAMS_WEBHOOK }}" \
            -d "{\"text\":\"CI failed: ${{ github.event.workflow_run.html_url }}\"}"
```

Trigger `Monitoring demo` again; watch `Notify on failure` fire on a red outcome.

> **Alerting design rule (guide §6):** alert on *negative transitions* only. A failing run is news; a green run is not — don't wire an alert for every completion or your team will learn to ignore it.

## Step 8 · Validate with Copilot

> "I have a GitHub Actions run where the `test` job's `payments` matrix cell failed intermittently. Walk me through: reading the run summary and job graph to confirm scope, re-running only the failed jobs, enabling step debug logging, and the exact shape of a `workflow_run` webhook that posts a Teams card when any workflow fails."

Check Copilot correctly separates *re-run failed jobs* from *re-run all*, mentions matrix-cell reruns, and includes the notification layer rather than only UI clicks.

---

## Expected outcome

- You can read any run: what failed, what it blocked, what still ran.
- You've re-run failed matrix cells and re-run from a failed step.
- You've used (then disabled) debug logging and downloaded logs for grep-able post-mortems.
- A `workflow_run` notifier alerts you when `Monitoring demo` fails.

## Key takeaways

- **The answer is in the tail of the failing step** — never read 2,000 lines.
- **Grey ≠ red** — cancelled runs are stopped by a human or a timeout, not a failure.
- **Re-run failed jobs is the default** recovery move; keep `fail-fast: false` while debugging.

## Troubleshooting

| Symptom | Fix |
|---|---|
| All matrix cells fail every time | The `RANDOM` trick is being too unlucky — or you forgot `fail-fast: false`; re-trigger a couple of runs |
| "Re-run failed jobs" greyed out | Only available on failed/cancelled runs; on success you can only re-run all |
| `Notify on failure` never fires | `workflow_run` must be in the *same repo* and watch the exact workflow name; check `conclusion` handling |
| Log looks identical with debug on | Verify *Step debug logging* (not just runner logging) is enabled in Settings → Actions |
| Secrets masked as `***` in logs | Workflow referenced a secret — review Modules 06/15; do not leave debug logging on |

## What's next

Reading and re-running gets you 80% of the way. **Lab 17** adds the Copilot muscle: hand the failing log to Copilot Chat and get back a diagnosis *and* a PR-ready fix.