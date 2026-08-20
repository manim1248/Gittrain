# Lab 18 · Advanced Authoring — Dynamic Matrices & the GitHub API

> **Confidential · Stalwart Learning**
> Module 18 · Guided lab · Session 4
> Companion: `guides/module-18-advanced-authoring.md` · Visualization: `module-18-visualization.html`

| | |
|---|---|
| **Objective** | Make a pipeline that *adapts*: a `discover` job computes which services changed and expands a build matrix with `fromJSON`; then use the GitHub API from a workflow to read PRs/tags and emit release metadata — the patterns you'll build on in Lab 19 |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Setups 02, 03, 04. Labs 09–10 workflows green. `ci-demo` should have 2+ microservices (e.g. `services/orders`, `services/payments`) — create a second service if it only has one. A repository with at least one merged PR (so the API returns data) |
| **Files you create** | `.github/scripts/changed_services.py`, `.github/workflows/dynamic-build.yml`, `.github/workflows/release-meta.yml` |

---

## Step 1 · Static vs dynamic

Open visualization **18.1** and click each pattern. The core idea that unlocks the whole module:

> Expressions are evaluated **when the run executes**, not when the file is committed. `github.*`, `needs.*.outputs`, `vars`, `secrets`, and `matrix` are *live* values.

## Step 2 · The `discover → fromJSON` pattern

Open visualization **18.2** to see the shape, then build it. First, the generator script — `.github/scripts/changed_services.py`:

```python
#!/usr/bin/env python3
"""Emit a JSON matrix of services whose files changed between two refs."""
import json
import subprocess
import sys

base, head = sys.argv[1], sys.argv[2]
out = subprocess.run(
    ["git", "diff", "--name-only", f"{base}...{head}"],
    capture_output=True, text=True,
).stdout

services = sorted({
    line.split("/")[1]
    for line in out.splitlines()
    if line.startswith("services/") and "/" in line.split("services/", 1)[1]
})
matrix = [{"service": s} for s in services] or [{"service": "all"}]
print(json.dumps({"include": matrix}))
```

Now the workflow — `.github/workflows/dynamic-build.yml`:

```yaml
name: Dynamic build
on:
  push:
    branches: [main]
    paths:
      - "services/**"

permissions:
  contents: read

jobs:
  discover:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.gen.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # need history for git diff
      - id: gen
        run: |
          echo "matrix=$(python3 .github/scripts/changed_services.py origin/main HEAD)" >> "$GITHUB_OUTPUT"

  build:
    needs: discover
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix: ${{ fromJSON(needs.discover.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v4
      - name: Build service
        run: echo "building ${{ matrix.service }}"
      - name: Test service
        run: echo "testing ${{ matrix.service }}"
```

> **Generate vs reuse (guide §3):** a reusable workflow (Module 10) parameterises at *run time* for the common path. This pattern *computes the matrix* from the repo — the pipeline grows itself when you add a service to `services/`.

## Step 3 · Validate the JSON before you wire it

Open visualization **18.3** and toggle services/platforms to internalise the cartesian math. Then prove the generator emits valid JSON **outside the workflow** first — this is the #1 `fromJSON` pitfall:

```bash
cd ci-demo
python3 .github/scripts/changed_services.py origin/main HEAD
# {"include": [{"service": "orders"}, {"service": "payments"}]}
```

Now touch a service and push:

```bash
echo "# change" >> services/orders/README.md
git add -A && git commit -m "lab18: trigger dynamic build" && git push
```

Open the run — the **job graph** should show `discover` followed by one `build` cell **per changed service** (Module 16 graph-reading applies here).

## Step 4 · Call the GitHub API — pick the right tool

Open visualization **18.4** and click each tool — `gh api`, `github-script`, `gh pr/release`, `curl` — to see the decision rule. Then build `.github/workflows/release-meta.yml` using **`actions/github-script`** (the choice when you have *logic* beyond one call):

```yaml
name: Release metadata
on:
  push:
    tags: ["v*"]

permissions:
  contents: read
  issues: read

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        id: meta
        with:
          script: |
            const { data: prs } = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: "closed",
              base: "main",
              per_page: 50,
            });
            const merged = prs.filter(p => p.merged_at);
            const notes = merged.map(p => `- ${p.title} (#${p.number})`).join("\n");
            core.setOutput("notes", notes);
            core.setOutput("count", String(merged.length));
      - name: Show release notes
        run: |
          echo "Merged PRs since last release: ${{ steps.meta.outputs.count }}"
          echo "${{ steps.meta.outputs.notes }}"
```

Push a tag (`git tag v0.1.0 && git push origin v0.1.0`) and watch the step compute the release notes from the API.

> **ADO mapping (guide §3):** ADO's REST API existed but you almost never called it *from* the pipeline. In Actions, `gh` and `github-script` are first-class citizen steps with the run's own token — the API is part of the workflow language.

## Step 5 · API-driven decision in the build path

Use a **`gh api` one-liner** (the right tool for a single read) to make a decision. Add a step to `dynamic-build.yml` that reads the latest tag and logs the next patch version:

```yaml
  release-check:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Compute next patch version
        run: |
          LATEST=$(gh api repos/${{ github.repository }}/tags --jq '.[0].name' || echo "v0.0.0")
          echo "latest tag: $LATEST"
          echo "next patch: ${LATEST%.*}.$(( ${LATEST##*.} + 1 ))"
```

> **Guardrail (Module 15):** give API jobs only the scopes they need — `contents: read` / `issues: read`, never blanket `write`. And don't call the API for data you already have in `github` context or `needs` outputs.

## Step 6 · Validate with Copilot

> "Refactor dynamic-build.yml: read services from `services/` via a `discover` job emitting a JSON matrix, build each with `fromJSON`, and add a `gh api` step that computes the next patch version from the latest tag. Keep permissions least-privilege, pin actions to SHAs, and don't call the API for data already in the github context."

Check: does Copilot keep `outputs:` and `fromJSON` in the right places, keep `permissions:` narrow, and *explain* why this beats a reusable workflow for this shape?

---

## Expected outcome

- A `discover` job whose JSON matrix drives N parallel build cells — adding a service requires zero workflow edits.
- A `github-script` step emitting release notes computed from the API.
- A `gh api` step reading tags at run time.
- You can articulate *when* to generate vs reuse (Module 10).

## Key takeaways

- **Validate the JSON before `fromJSON`** — malformed JSON errors the whole strategy block.
- **Pick the API tool by need:** `gh api` for one read, `github-script` for logic, `gh pr/release` for automation, `curl` only as an escape hatch.
- **API calls inherit the token's permissions** — keep them narrow (Module 15).

## Troubleshooting

| Symptom | Fix |
|---|---|
| `fromJSON` fails: "Unexpected token" | The output isn't valid JSON — run the generator standalone (Step 3); ensure no log noise on the output line |
| Matrix shows `{service: all}` | Your change wasn't under `services/`, or the diff range is wrong — check `git diff origin/main...HEAD` |
| Empty `notes` output | No merged PRs base `main` — the API returns an empty list; create/merge a throwaway PR |
| `gh api` 403 | Job lacks the scope — add `contents: read` / `issues: read` to that job's `permissions:` (Module 15) |
| `discover` output empty | `fetch-depth: 0` missing → shallow clone has no history for the diff |

## What's next

These patterns are the raw material for **Lab 19** — the end-to-end capstone that chains trigger → discover → matrix build → push → OIDC → environment-gated GitOps hand-off.