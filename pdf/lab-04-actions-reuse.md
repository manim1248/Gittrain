# Lab 04 · Actions, Marketplace & Reuse

> **Confidential · Stalwart Learning**
> Module 04 · Guided lab · Session 1
> Companion: `guides/module-04-actions-marketplace-reuse.md` · Visualization: `module-04-visualization.html`

| | |
|---|---|
| **Objective** | Consume Marketplace actions safely, then author both reuse mechanisms — a **composite action** and a **reusable workflow** — and call them from a workflow |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Lab 03 complete (you can reuse its pipeline) |
| **Files you create** | `actions/build-node/action.yml`, `.github/workflows/consumer.yml`, `.github/workflows/shared-ci.yml` |

---

## Step 1 · Know your actions — and pin them

All Labs so far used first-party actions (`actions/*`). In this step, check what you're pulling in and how to pin it:

1. Open `module-04-visualization.html` **4.1 Pinning policy** — toggle SHA / tag / branch and read the security meter.
2. On github.com, open `actions/upload-artifact` → the **Releases** tab shows the current major (v4) and its latest full **commit SHA**.
3. Your production policy (preview of Session 3, Module 15): pin **third-party** actions by full SHA with the tag in a comment, e.g.

```yaml
- uses: some-org/some-action@<full-40-char-sha>   # v1.2.3
```

For this sandbox we keep `@v4` tags for readability — but from now on, every workflow gets an explicit **least-privilege `permissions:` block** (you already added these in Labs 01–03).

> Run the **4.3 Supply-chain scorecard** in the visualization now and note which items your current workflows satisfy (most, after the earlier labs).

## Step 2 · Write a composite action

A composite action bundles steps you repeat. Create `actions/build-node/action.yml`:

```yaml
name: Build Node service
description: Check out, install and build a Node 20 service
inputs:
  build-cmd:
    description: Build command to run
    required: false
    default: npm run build
outputs:
  status:
    description: build exit code
    value: ${{ steps.do-build.outputs.status }}
runs:
  using: composite
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
    - id: do-build
      run: |
        ${{ inputs.build-cmd }}
        echo "status=$?" >> "$GITHUB_OUTPUT"
      shell: bash          # ← required on every composite run: step
```

> Composite gotchas to observe: every `run:` needs `shell:`; no top-level `env:`/`if:`; inputs/outputs replace the full contexts.

## Step 3 · Use the composite action

Create `.github/workflows/consumer.yml`:

```yaml
name: Consumer
on: [workflow_dispatch]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build with local composite
        uses: ./actions/build-node
        with:
          build-cmd: npm init -y && npm pkg set scripts.build="echo built" && npm run build
      - name: Show composite output
        run: echo "build status = ${{ steps.build.outputs.status }}"
```

Wait — `steps.build` only works if the step has an `id:`. Add it:

```yaml
      - id: build
        name: Build with local composite
        uses: ./actions/build-node
```

Run it. The composite's three steps appear expanded in the run UI (checkout → setup-node → do-build).

## Step 4 · Write and call a reusable workflow

A reusable workflow shares a *whole job* via `on: workflow_call`. Create `.github/workflows/shared-ci.yml`:

```yaml
name: Shared CI
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
        required: false
    secrets:
      WELCOME:
        required: false
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: echo "Shared build on node ${{ inputs.node-version }}"
      - name: Optional greeting from secret
        if: env.WELCOME != ''          # secrets aren't allowed in if: — route via env
        env:
          WELCOME: ${{ secrets.WELCOME }}
        run: echo "greeting: $WELCOME"
```

Call it from a second workflow, `consumer-2.yml`:

```yaml
name: Consumer 2
on: [workflow_dispatch]
permissions:
  contents: read
jobs:
  ci:
    uses: ./.github/workflows/shared-ci.yml      # same-repo reference
    with:
      node-version: '22'
    secrets: inherit                             # forwards repo secrets
```

Run `Consumer 2`. The Actions tab shows the `build` job **inside** the caller's run — the callee's jobs became your jobs.

> **Org-level reuse (preview of Session 2, Module 10):** the same pattern works across repos — point `uses:` at `org/shared-workflows/.github/workflows/shared-ci.yml@v1` and version it by ref. For this lab, same-repo is enough to learn the mechanics.

## Step 5 · Composite vs reusable — apply the decision

Open `module-04-visualization.html` **4.2**. Answer the question *"What do I want to share?"*:

- A **fixed step recipe** → composite action (you did this in Step 3).
- A **whole pipeline with inputs & secrets** called by many repos → reusable workflow (Step 4).

Reflect: in your `ci-demo`, which mechanism would you use to standardise CI across *all* your microservice repos? (Answer: the reusable workflow — it's the pattern Session 2, Module 10 builds out.)

## Step 6 · Supply-chain review of your own work

Run the **4.3 scorecard** again and fix any gaps on `consumer.yml`/`consumer-2.yml`:
- explicit `permissions:` block (done), pins readable, actions from trusted publishers (`actions/*`, local `./`), secrets scoped (`secrets: inherit` is OK here because the callee is yours — reconsider for third-party callees).

---

## Expected outcome

- A working **composite action** used by a workflow, with its inputs and outputs flowing correctly.
- A **reusable workflow** called with `with:` + `secrets: inherit`, and the callee's jobs visible in the caller's run.
- You can state when to reach for each mechanism and can defend your pinning policy.

## Key takeaways

- **Every action is third-party code running in your CI** — pin it (SHA + tag comment), restrict `permissions:`, prefer verified publishers.
- Composite action = share **steps**; reusable workflow = share **jobs/pipeline**. Both are versioned by ref.
- `secrets: inherit` is convenient but forwards *everything* — scope it when the callee is lower-trust.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Composite `run:` fails with "unrecognized named-value" | Missing `shell:` on the `run:` step — add `shell: bash` |
| Composite can't find `actions/checkout` | It can't use `./` paths for actions; `actions/*` refs are fine — checkout runs from the runner's context |
| Reusable workflow "is not reusable" error | The callee must declare `on: workflow_call` and be in `.github/workflows/` |
| Caller can't override callee jobs | By design — version by ref and design the input contract up front |
| `${{ steps.build.outputs.status }}` empty | Step needs `id: build` AND the composite must declare `outputs:` bound to a step `id` |