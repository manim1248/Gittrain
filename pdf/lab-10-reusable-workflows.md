# Lab 10 · Standardising Pipelines: Reusable Workflows & Composite Actions

> **Confidential · Stalwart Learning**
> Module 10 · Guided lab · Session 2
> Companion: `guides/module-10-reusable-workflows-composite-actions.md` · Visualization: `module-10-visualization.html`

| | |
|---|---|
| **Objective** | Stop copying CI YAML per service. Extract one **reusable workflow** (`workflow_call`) with a strict input/secret contract, factor its build-push steps into a **composite action**, convert the per-service workflows into thin callers, and version by ref — with Copilot doing the extraction |
| **Time** | ~65 min (guided) |
| **Prerequisites** | Lab 09 complete (Docker Hub build/push green for `checkout`) |
| **Files you create** | `.github/workflows/shared-ci.yml`, `actions/build-push/action.yml`, rewritten `checkout-ci.yml` / `orders-ci.yml` |

---

## Step 1 · Inventory the drift (the migration playbook, step 1)

You now have two service workflows that *should* be identical — and aren't:

1. `checkout-ci.yml` (Lab 07) builds but **doesn't push** an image.
2. `ci.yml` (Lab 07) has the detection + matrix; `build-push.yml` (Lab 09) pushes `checkout` but there's no `orders` equivalent.

Run the guide's inventory question: *list every variation across the copies* — node version, dockerfile path, cache path, image name, push gating. In a 5-service fleet this drift is exactly what Module 10 removes (guide §1). Open visualization **10.1 Drift explorer**: switch `checkout`/`orders`/`payments` and compare the before/after code — your files are the "before".

## Step 2 · Design the contract (playbook step 2)

Every variation becomes an **input**; credentials stay **secrets**; the produced tag is an **output**:

| Input / secret / output | What varies | Example |
|---|---|---|
| `service` (input, required) | directory under `services/` | `checkout` |
| `dockerfile` (input, required) | Dockerfile path | `services/checkout/Dockerfile` |
| `node-version` (input, default `20`) | toolchain version | `'20'` |
| `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` (secrets, required) | registry credentials | set at caller |
| `image-tag` (output) | what the caller hands to Flux | `sha-<7>` tags |

> Design rule (guide §3): inputs are the **only** way the caller varies the shared pipeline — there's no way to inject custom steps. Don't under-specify it, or drift creeps back in.

## Step 3 · The composite action — the fixed "build & push" recipe

Not everything should be a whole workflow. The build+tag+push steps are the same everywhere, so package them as a **composite action** in the same repo (guide §7):

Create `actions/build-push/action.yml`:

```yaml
name: 'Build & Push'
description: 'Build, tag and push a service image to Docker Hub (Buildx + metadata + gha cache)'
inputs:
  service:
    description: 'Service directory under services/'
    required: true
  dockerfile:
    description: 'Path to the Dockerfile'
    required: true
  registry-username:
    description: 'Docker Hub username'
    required: true
  registry-token:
    description: 'Docker Hub access token'
    required: true
outputs:
  tags:
    description: 'Generated image tags'
    value: ${{ steps.meta.outputs.tags }}
runs:
  using: 'composite'
  steps:
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        username: ${{ inputs.registry-username }}
        password: ${{ inputs.registry-token }}
    - uses: docker/metadata-action@v5
      id: meta
      with:
        images: ${{ inputs.registry-username }}/${{ inputs.service }}
        tags: |
          type=ref,event=branch
          type=sha,format=short
    - uses: docker/build-push-action@v6
      with:
        context: services/${{ inputs.service }}
        file: ${{ inputs.dockerfile }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

> Gotchas you just avoided (guide §7): composite `run:` steps would need `shell:` (these are all action steps, so not required); the composite can't see the caller's `env:` — everything came in as **inputs**; and note `push: true` here — the *caller* controls when the workflow runs (Step 5).

## Step 4 · The reusable workflow — `on: workflow_call`

Create `.github/workflows/shared-ci.yml` — the one definition every service calls:

```yaml
name: Shared CI
on:
  workflow_call:
    inputs:
      service:
        type: string
        required: true
        description: 'Service directory under services/, e.g. checkout'
      dockerfile:
        type: string
        required: true
      node-version:
        type: string
        default: '20'
    outputs:
      image-tag:
        description: 'Immutable image tags built for this run'
        value: ${{ jobs.build.outputs.image-tag }}
    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      image-tag: ${{ steps.build-push.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
          cache-dependency-path: services/${{ inputs.service }}/package-lock.json
      - run: npm ci --prefix services/${{ inputs.service }}
      - run: npm test --prefix services/${{ inputs.service }}
      - name: Build & push ${{ inputs.service }}
        id: build-push
        uses: ./actions/build-push
        with:
          service: ${{ inputs.service }}
          dockerfile: ${{ inputs.dockerfile }}
          registry-username: ${{ secrets.DOCKERHUB_USERNAME }}
          registry-token: ${{ secrets.DOCKERHUB_TOKEN }}
```

Notice (guide §3 design rules): the callee **owns** `permissions:` and the job structure; secrets are **explicitly listed** (not `inherit`); `node-version` is the only variation passed through; the output flows back to the caller.

> **ADO mapping:** `workflow_call` ≈ ADO **YAML template** (`extends:`) — but with a *harder* contract: ADO templates merge YAML into the caller, GitHub executes a fixed workflow and only passes data via `with:`/`secrets:`.

## Step 5 · Convert the per-service workflows into thin callers

Rewrite `.github/workflows/checkout-ci.yml` (same repo → relative `uses:`):

```yaml
name: Checkout CI
on:
  push:
    branches: [main]
    paths: ['services/checkout/**']
  pull_request:
    paths: ['services/checkout/**']
  workflow_dispatch:

jobs:
  shared-ci:
    uses: ./.github/workflows/shared-ci.yml
    with:
      service: checkout
      dockerfile: services/checkout/Dockerfile
      node-version: '20'
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

Create the matching `.github/workflows/orders-ci.yml` for `orders` (same caller, `service: orders`, `dockerfile: services/orders/Dockerfile` — you'll need the Lab 09 `app.js`/`Dockerfile` in `services/orders` too, or delete the `build-push` step impact by keeping the service minimal).

Then delete the drift: remove the old `ci.yml` detection+matrix job and `build-push.yml` — their job is done by one shared definition. The per-service workflow is now ~15 lines: trigger + `uses:` + `with:` + `secrets:`.

> **Same-repo vs cross-repo (guide §3/§6):** within `ci-demo` you use the relative `uses: ./.github/workflows/shared-ci.yml`. For the production pattern (guide §1), the shared workflow lives in `org/shared-workflows` and callers use `uses: myorg/shared-workflows/.github/workflows/ci.yml@v1` — pinned by **ref**.

## Step 6 · Prove the abstraction

1. Push a change to `services/checkout` → **Checkout CI** runs the shared workflow; an image `:main`/`:sha-<7>` appears on Docker Hub.
2. Push a change to `services/orders` → **Orders CI** does the same, from the *same* `shared-ci.yml`.
3. Edit `shared-ci.yml` (e.g. add a `timeout-minutes` or a new test step) and push — **every caller** picks it up on the next run. That's the win **and** the risk: a change in one place deploys everywhere (guide §3 versioning).

## Step 7 · Versioning by ref + the `secrets: inherit` decision

Open visualization **10.3 Contract & caller validation** and click the three caller buttons — see which callers pass and which fail validation (missing `dockerfile`, invalid `choice`), exactly what `workflow_call` enforces for you.

Then two decisions for production:

1. **Versioning:** pin `uses:` refs — `@v1` for stable, `@main` for canary, a full **SHA** for audit-pinned rollouts (guide §3). Same-repo relative calls always track the current branch, so pinning matters once you move to `org/shared-workflows`.
2. **Secrets:** we used an **explicit list** — deliberate and auditable. `secrets: inherit` forwards *every* org/repo secret the caller can access; only switch to it once the shared workflow is trusted (guide §5).

## Step 8 · Validate with Copilot (the extraction workflow)

Run these three prompts (guide §8), then apply the Module 05 guardrails — read the output, confirm secrets/inputs are only *read*, verify the `uses:` path resolves, and dry-run on a branch before promoting:

> **1 — extract:** "Compare these two CI workflows (checkout-ci.yml and orders-ci.yml). They're per-service copies that have drifted. Design a single reusable workflow with `on: workflow_call`. List every variation as an `input`, declare `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` in `secrets:`, and expose the image tags as an `output`. Keep buildx + metadata + build-push with `type=gha` cache."
>
> **2 — generate a caller:** "Generate a caller for `services/orders`: trigger on push to `main` + pull_request with `paths: ['services/orders/**']`, one job that `uses: ./.github/workflows/shared-ci.yml` with `with: { service: orders, dockerfile: services/orders/Dockerfile }` and the two Docker Hub secrets."
>
> **3 — review the drift removed:** "Given the shared workflow above, tell me which per-service files are now redundant, and flag anything in the shared workflow that still assumes a specific service."

Watch for Copilot's known blind spots (guide §8): omitting `secrets:` in `workflow_call`, floating `uses:` refs, forgetting the callee can't use caller `env:`.

---

## Expected outcome

- One `shared-ci.yml` called by two thin callers; both services build, test, and push to Docker Hub from one definition.
- A reusable **composite action** (`actions/build-push`) doing the fixed build/tag/push recipe inside the shared workflow.
- You can articulate: *reusable workflow = share a pipeline; composite action = share a step recipe; `run:` = share nothing* (guide §2).

## Key takeaways

- **The migration end-state:** N service repos + 1 versioned shared workflow + thin callers — no drift.
- **Inputs are the contract**; secrets are explicit; the callee owns permissions and job structure.
- **Version by ref** (`@v1`/SHA); composite actions need everything passed as inputs (no caller `env:`).

## Troubleshooting

| Symptom | Fix |
|---|---|
| `uses: ./.github/workflows/shared-ci.yml` not found | The relative path is **`./.github/workflows/…`** and the file must be committed on the current branch |
| `Missing required input: dockerfile` at run time | The caller's `with:` must include every `required: true` input — `workflow_call` validation is strict (Step 7) |
| Secrets empty inside the callee | The caller must list them in `secrets:` (or use `secrets: inherit`) — the callee reads *declared* names only |
| "Action not allowed" / permission errors | The shared workflow owns `permissions:` — callers can't add them; fix inside `shared-ci.yml` |
| Composite step ignores `timeout-minutes`/`if:` | Composites can't take job-level `if:`/`timeout` on the whole `uses:` step — put such controls on the *caller's job* or *inside* the composite's own steps |
| Callers changed but behavior identical | You edited a copy, not the shared file — after Step 5 there should be **one** `shared-ci.yml` and only thin callers |