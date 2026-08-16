# Lab 02 · Events & Triggers — build vs release triggers

> **Confidential · Stalwart Learning**
> Module 02 · Guided lab · Session 1
> Companion: `guides/module-02-events-and-triggers.md` · Visualization: `module-02-visualization.html`

| | |
|---|---|
| **Objective** | Make one workflow respond to push, pull_request, schedule, workflow_dispatch and repository_dispatch, and choose triggers for a build vs a release workflow |
| **Time** | ~45 min (guided) |
| **Prerequisites** | Lab 01 complete (`ci-demo` green on push) |
| **Files you create** | `.github/workflows/triggers.yml`, `.github/workflows/release.yml` |

---

## Step 1 · One workflow, five events

Create `.github/workflows/triggers.yml`:

```yaml
name: Triggers
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize]
  schedule:
    - cron: '*/30 * * * *'        # every 30 min — see Step 4
  workflow_dispatch:
    inputs:
      target:
        description: Where to target
        type: choice
        options: [dev, staging, production]
        default: dev
      dry_run:
        description: Dry run only
        type: boolean
        default: false
  repository_dispatch:
    types: [build-requested]

jobs:
  inspect:
    runs-on: ubuntu-latest
    steps:
      - name: What fired this run?
        run: |
          echo "event        = ${{ github.event_name }}"
          echo "ref          = ${{ github.ref }}"
          echo "head_ref     = ${{ github.head_ref }}"
          echo "base_ref     = ${{ github.base_ref }}"
          echo "sha          = ${{ github.sha }}"
          echo "actor        = ${{ github.actor }}"
      - name: workflow_dispatch inputs
        if: github.event_name == 'workflow_dispatch'
        run: |
          echo "target  = ${{ github.event.inputs.target }}"
          echo "dry_run = ${{ github.event.inputs.dry_run }}"
      - name: repository_dispatch payload
        if: github.event_name == 'repository_dispatch'
        run: echo "client payload = ${{ toJSON(github.event.client_payload) }}"
```

Commit + push. You already have a `push` run in the Actions tab — open `Triggers` and check the `inspect` job output.

## Step 2 · Trigger it with a pull request

```bash
git checkout -b feature/lab02
echo "trigger-lab" > pr-test.txt
git add pr-test.txt
git commit -m "lab02: open a PR"
git push -u origin feature/lab02
```

Open a **pull request** (from the branch banner, or `github.com/<you>/ci-demo/pull/new/feature/lab02`).

1. The PR shows the `Triggers` run (event `pull_request`, `head_ref=feature/lab02`, `base_ref=main`).
2. Push another commit to the branch — a new run fires (the `synchronize` type).
3. Note the run's `ref` is the **merge ref** (`refs/pull/N/merge`) — GitHub validates the PR *merged* into base.
4. Merge the PR and watch the `push` run on `main`.

## Step 3 · Manual dispatch with inputs

**Actions → Triggers → Run workflow** → choose `production` and tick *dry run* → run. Check the output shows `target=production dry_run=true`.

> Recall from the guide: `workflow_dispatch` inputs arrive **string-typed** in `github.event.inputs.*` — `"true"`, not `true`.

## Step 4 · Schedule — verify the cron math, then disable

The `*/30 * * * *` cron will fire every 30 minutes. Confirm with the **cron visualiser** in `module-02-visualization.html` (section 2.3): paste `*/30 * * * *` and see the next runs in UTC.

Then, since nobody wants 30-minute runs forever, change the cron to a daily one (or remove `schedule`):

```yaml
  schedule:
    - cron: '0 2 * * *'   # daily 02:00 UTC
```

Notes to carry forward: schedule runs on the **default branch only**, is **best-effort**, and delivers **no payload**.

## Step 5 · repository_dispatch from the API

Dispatch an event with `curl` (a GitHub token with repo scope, e.g. a fine-grained PAT, or `gh`):

```bash
gh api -X POST repos/<you>/ci-demo/dispatches \
  -f event_type=build-requested \
  -f 'client_payload[trigger]=curl'
```

Or with curl:

```bash
curl -X POST https://api.github.com/repos/<you>/ci-demo/dispatches \
  -H "Authorization: Bearer $GH_TOKEN" \
  -d '{"event_type":"build-requested","client_payload":{"trigger":"curl"}}'
```

1. Watch a new `Triggers` run appear.
2. Open the `repository_dispatch payload` step — it prints the JSON you sent.

> Security note from the guide: `repository_dispatch` **cannot carry secrets**. Use repo/org secrets for anything sensitive.

## Step 6 · Build vs release — apply the decision flow

Open `module-02-visualization.html` section 2.2 and answer: *"What is this workflow for?"* → **release**; *"How is the change created?"* → **direct push / merged to main**; *"Do you need humans?"* → **manual**.

The tool recommends a manual trigger. Implement it — create `.github/workflows/release.yml`:

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        default: staging
jobs:
  prepare:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Preparing release for ${{ github.event.inputs.environment }}"
```

Run it once manually. You now have the **build trigger** (`triggers.yml`) and the **release trigger** (`release.yml`) pattern you'll extend in Session 2–3.

---

## Expected outcome

- `triggers.yml` observed firing under **4 different events** (push, pull_request, workflow_dispatch, repository_dispatch) — plus a scheduled one.
- A `release.yml` gated behind a manual `workflow_dispatch` with an environment input.
- You can justify every trigger choice against the build-vs-release decision flow.

## Key takeaways

- **Events decide when the workflow runs**; the `github` context tells you *which* event and *why*.
- PR runs use the **merge ref**; schedule runs are **best-effort on default branch**; `workflow_dispatch` is the reliable manual/release gate; `repository_dispatch` is the external-systems entry point (no secrets).
- Decide **build vs release** up front — don't make a release workflow fire on every push.

## Troubleshooting

| Symptom | Fix |
|---|---|
| PR run has no `push` values | Correct — the payload for `pull_request` uses `head_ref`/`base_ref`; `github.ref` is the merge ref |
| Dispatch curl returns 404 | You need `repo`/`actions` write scope on the token; URL is `/repos/<owner>/<repo>/dispatches` |
| Schedule run didn't appear | Schedules fire within **minutes** of the cron, best-effort, on the default branch only — wait, then check the Actions log filter |
| `toJSON` prints nothing | `client_payload` is empty if you sent no payload — re-send with `client_payload` |