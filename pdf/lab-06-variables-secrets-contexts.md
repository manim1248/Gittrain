# Lab 06 · Variables, Secrets & Contexts

> **Confidential · Stalwart Learning**
> Module 06 · Guided lab · Session 2
> Companion: `guides/module-06-variables-secrets-contexts.md` · Visualization: `module-06-visualization.html`

| | |
|---|---|
| **Objective** | Store configuration in the right place (workflow `env:`, repo/org/environment **variables** and **secrets**), read it back via the `vars`/`secrets`/`github`/`env` contexts, and make values survive across steps with `$GITHUB_ENV` |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Labs 01–05 complete · Setup 6 (Docker Hub secrets) recommended |
| **Files you create** | `.github/workflows/contexts.yml`, `variables.yml`, `secrets.yml`, `env-scope.yml`, `githubenv.yml` |

---

## Step 1 · The context inspector — what GitHub injects into every run

Create `.github/workflows/contexts.yml`:

```yaml
name: Contexts
on: [push, workflow_dispatch]

jobs:
  inspect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Expression contexts
        run: |
          echo "event_name  = ${{ github.event_name }}"
          echo "ref         = ${{ github.ref }}"
          echo "sha         = ${{ github.sha }}"
          echo "run_number  = ${{ github.run_number }}"
          echo "repository  = ${{ github.repository }}"
          echo "actor       = ${{ github.actor }}"
          echo "workspace   = ${{ github.workspace }}"
          echo "runner.os   = ${{ runner.os }}"
          echo "runner.arch = ${{ runner.arch }}"
      - name: Predefined shell env vars
        run: |
          echo "GITHUB_SHA    = $GITHUB_SHA"
          echo "GITHUB_RUN_ID = $GITHUB_RUN_ID"
          echo "GITHUB_ACTOR  = $GITHUB_ACTOR"
          echo "RUNNER_OS     = $RUNNER_OS"
          echo "CI            = $CI"
```

Commit + push, then **Actions → Contexts**:

1. Open the run, expand the two steps, and map each value to the guide §6 table: `github.ref` = the branch you pushed, `github.run_number` = the run counter, `runner.os` = `Linux`.
2. Re-run with **Run workflow** (*workflow_dispatch*). Compare `event_name` (`push` → `workflow_dispatch`) — same workflow, different injected data.
3. In `module-06-visualization.html`, section **6.1 Expression resolver**, click the `github.ref`, `github.sha`, `github.actor` chips and confirm they match what you just saw.

> **ADO mapping to note:** `${{ github.run_number }}` ≈ `$(Build.BuildNumber)`; `${{ github.sha }}` ≈ `$(Build.SourceVersion)`; `${{ github.workspace }}` ≈ `$(System.DefaultWorkingDirectory)`.

## Step 2 · Variables (`vars`) — repo, org, and precedence

Variables are **non-secret** config stored in settings. Set three scopes with the `gh` CLI (or the Settings UI):

```bash
gh variable set IMAGE_REGISTRY --repo <you>/ci-demo --body ghcr.io
gh variable set PREFERENCE     --org <your-org> --body org-level --visibility all
gh variable set PREFERENCE     --repo <you>/ci-demo --body repo-level
gh variable list --repo <you>/ci-demo
```

(If you don't have an org, create `PREFERENCE` only at repo level and skip the org line — the precedence demo still works with repo + environment below.)

```bash
gh variable set DEPLOY_ENV --env dev --repo <you>/ci-demo --body dev-scope
```

Now read them in a workflow — `.github/workflows/variables.yml`:

```yaml
name: Variables
on: [push, workflow_dispatch]

env:
  REGISTRY:  ${{ vars.IMAGE_REGISTRY }}
  PREFERENCE: ${{ vars.PREFERENCE }}

jobs:
  show:
    runs-on: ubuntu-latest
    steps:
      - run: echo "REGISTRY=$REGISTRY PREFERENCE=$PREFERENCE"
      - run: echo "vars.IMAGE_REGISTRY = ${{ vars.IMAGE_REGISTRY }}"
```

Push and open the run:

1. `PREFERENCE=repo-level` — when the same name exists at **repo and org**, the repo wins (guide §3: *env > repo > org*).
2. Delete the repo-level variable (`gh variable delete PREFERENCE --repo <you>/ci-demo`) and push again — the **org value** now surfaces. Re-create the repo variable afterwards.
3. Add `DEPLOY_ENV: ${{ vars.DEPLOY_ENV }}` to the job `env:` and a step that echoes it — confirm the **environment** scope value. Then open the visualization **6.4 Where secrets & variables live** and match each box to a scope you just used.

> Rules you just exercised: `vars` are plaintext (visible to anyone with repo read access — **never** put a password in a variable), set in UI/CLI **not** in YAML, and override org < repo < environment.

## Step 3 · Secrets — the masking demo (with a dummy value)

First add a **dummy** secret so you can watch masking safely:

```bash
gh secret set TEST_SECRET --repo <you>/ci-demo --body "super-secret-value-123"
```

(Setup 6 already created `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN` — leave those alone.)

Create `.github/workflows/secrets.yml`:

```yaml
name: Secrets
on: [push, workflow_dispatch]

jobs:
  demo:
    runs-on: ubuntu-latest
    steps:
      - name: Masking demo — never do this with a real secret
        run: echo "TEST_SECRET = ${{ secrets.TEST_SECRET }}"
      - name: Guard against an unset secret (empty string)
        run: |
          if [ "$HAS_TOKEN" = "true" ]; then
            echo "DOCKERHUB_TOKEN is configured"
          else
            echo "DOCKERHUB_TOKEN is EMPTY — run Setup 6"
          fi
        env:
          HAS_TOKEN: ${{ secrets.DOCKERHUB_TOKEN != '' }}
```

Push, open the run:

1. The `echo` step prints `TEST_SECRET = ***` — GitHub masked the value the moment it appeared in the log.
2. The guard prints `DOCKERHUB_TOKEN is configured` (or the empty-string warning). This is the **guide §4 gotcha**: an *unset* secret is an empty string, not an error.

> **Remember the pitfall from the guide:** masking only kicks in **after first reference**. Do **not** copy the masking demo into real workflows — echoing secrets is how they leak. Real workflows use secrets *only* inside `${{ secrets.X }}` in `with:`/`env:`/`run:` that consumes them (you'll do exactly that in Lab 09).

## Step 4 · Environment secrets — only resolve for the target environment

Environments tie values to a deploy target and (Session 3) approval gates. Create one now:

1. Repo **Settings → Environments → New environment → `dev`**.
2. On the `dev` environment page → **Environment secrets** → add `DEV_API_KEY` = `dev-key-abc` (any dummy value).
3. Also set a **variable** `DEPLOY_ENV` = `dev-scope` on that environment (or keep the Step 2 one).

Create `.github/workflows/environment.yml`:

```yaml
name: Environment
on:
  workflow_dispatch:

jobs:
  use-dev-secret:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - run: echo "DEV_API_KEY present: ${{ secrets.DEV_API_KEY != '' }}"
        # do NOT print the value — just whether it resolved
```

Trigger it with **Run workflow** and confirm `DEV_API_KEY present: true`. The key idea: `secrets.DEV_API_KEY` **only resolves when a run targets the `dev` environment** — remove `environment: dev` and it becomes `false`. That is exactly how you'll scope registry credentials to a deploy job in Session 3.

## Step 5 · The scope ladder — `env:` at workflow / job / step

Create `.github/workflows/env-scope.yml`:

```yaml
name: Env scope
on: [push, workflow_dispatch]

env:                                # workflow-level
  BUILD_MODE: release
  REGISTRY: ghcr.io

jobs:
  build:
    runs-on: ubuntu-latest
    env:                            # job-level (overrides for this job)
      REGISTRY: localhost:5000
    steps:
      - name: Job-level env
        run: echo "$BUILD_MODE $REGISTRY"      # release localhost:5000
      - name: Step-level env
        run: echo "$BUILD_MODE $REGISTRY"      # debug localhost:5000
        env:
          BUILD_MODE: debug
      - name: env context in expressions
        run: echo "${{ env.BUILD_MODE }} ${{ env.REGISTRY }}"
```

Push and read the three outputs — they confirm the ladder: step overrides job overrides workflow. Open visualization **6.3 Scope ladder** and click the three chips; the resolved table should match your run.

> **ADO mapping:** workflow-level `env:` ≈ pipeline variables; job-level `env:` ≈ per-stage variables; step-level `env:` ≈ per-task variables.

## Step 6 · `$GITHUB_ENV` — make a value survive across steps

Each `run:` step is its own shell — `export` dies with the step. `$GITHUB_ENV` persists. Create `.github/workflows/githubenv.yml`:

```yaml
name: GITHUB_ENV
on: [push, workflow_dispatch]

jobs:
  persist:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1 — write to $GITHUB_ENV
        run: echo "IMAGE_TAG=sha-${GITHUB_SHA::7}" >> "$GITHUB_ENV"
      - name: Step 2 — plain export (dies with this step)
        run: export LOST_VAR="gone after this step"
      - name: Step 3 — what survived?
        run: |
          echo "IMAGE_TAG = $IMAGE_TAG"
          echo "LOST_VAR  = ${LOST_VAR:-<empty — the export died>}"
```

Push and read Step 3: `IMAGE_TAG` survived (it went through `$GITHUB_ENV`), `LOST_VAR` is empty. Run the visualization **6.5 $GITHUB_ENV — the persistence trick** — it replays this exact job.

> **ADO mapping:** `echo "FOO=bar" >> $GITHUB_ENV` ≈ `echo "##vso[task.setvariable variable=FOO]bar"`. You'll use `$GITHUB_ENV` (and `$GITHUB_OUTPUT`) in every later lab to carry image tags between steps.

## Step 7 · Copilot checkpoint

> "Create a workflow that computes an image tag from `${{ github.sha }}` via `$GITHUB_OUTPUT`, stores the registry in a repo **variable** `IMAGE_REGISTRY`, logs into a registry with `${{ secrets.DOCKERHUB_USERNAME }}` / `${{ secrets.DOCKERHUB_TOKEN }}`, and adds least-privilege `permissions: contents: read`."

Review the output: are all non-secret values in `vars`? Are secrets referenced but never printed? Is the tag passed by output, not by `env:`?

---

## Expected outcome

- You read `github.*`/`runner.*` contexts and can predict their values from the guide §6 table.
- Variables at repo/org/environment scope, with the *env > repo > org* precedence proven in a run.
- A dummy secret masked as `***`, an unset-secret guard, and an environment secret that only resolves for its environment.
- `env:` scope ladder and `$GITHUB_ENV` persistence both demonstrated with real runs.

## Key takeaways

- **Contexts are read-only data injected into the run** — `github`, `vars`, `secrets`, `env`, `needs`, `matrix`, `runner`.
- **`vars` for config, `secrets` for credentials** — set both in settings/CLI, never in YAML.
- **`env:` blocks scope to workflow/job/step**; `$GITHUB_ENV` is the only way a *computed* value outlives its step.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `vars.IMAGE_REGISTRY` empty | The variable must exist in Settings/CLI — check `gh variable list --repo <you>/ci-demo`; remember env > repo > org precedence |
| Secret prints the real value in logs | It was echoed **before** its first `${{ secrets.X }}` reference — masking is post-first-reference; never echo secrets |
| `secrets.DEV_API_KEY != ''` is `false` | The job must declare `environment: dev` — env secrets don't resolve otherwise |
| `$GITHUB_ENV` value missing in a later step | It persists to later steps **of the same job only**; jobs can't share `env:` — use job outputs (`needs.<id>.outputs`) instead |
| `gh` command not found / auth | `gh auth login` (Setup 3 sign-in also authorises `gh`); scope `repo`/`workflow` required |