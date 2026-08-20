# Lab 17 · Troubleshooting with GitHub Copilot — From Log to PR-Ready Fix

> **Confidential · Stalwart Learning**
> Module 17 · Guided lab · Session 4
> Companion: `guides/module-17-troubleshooting-with-copilot.md` · Visualization: `module-17-visualization.html`

| | |
|---|---|
| **Objective** | Take a real failing run, paste its log tail into **Copilot Chat** with the right context, get a diagnosis, then a PR-ready fix — and run the fix through the Module 15 review guardrails before merging |
| **Time** | ~45 min (guided) |
| **Prerequisites** | Setup 04 (Copilot). Setup 03 (VS Code) with the Copilot + GitHub Actions extensions. A failed run from Lab 16 (or any red workflow in `ci-demo`) |
| **Files you create** | `.github/workflows/troubleshoot-demo.yml` (a deliberately misconfigured workflow), then its fix commit |

---

## Step 1 · The troubleshooting loop

Open visualization **17.1** and click through all six steps. The loop you'll run repeatedly:

```
identify → copy log tail → prompt Copilot → diagnosis → fix as diff → review & verify
```

> **No ADO equivalent:** this is a Copilot-native workflow — the closest ADO analogue is a teammate you can paste a log to. The skill is in *giving it the context* a good teammate would ask for.

## Step 2 · Build a workflow with a genuine bug

Create `.github/workflows/troubleshoot-demo.yml`. This one has a *real* misconfiguration that fails in a famous way — **"Resource not accessible by integration"** (a `403`):

```yaml
name: Troubleshoot demo
on:
  workflow_dispatch:

permissions:
  contents: read          # too narrow for what the step tries to do

jobs:
  tag-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Try to create a git tag (fails — no write permission)
        run: |
          gh api --method POST \
            repos/${{ github.repository }}/git/refs \
            -f ref=refs/tags/demo-v1 \
            -f sha=${{ github.sha }}
```

Push and trigger the workflow. It will fail with a **403**. This is the exact failure class from visualization **17.2 (Token permission error)** — and one of the most common real-world Actions failures.

## Step 3 · Copy the log tail the right way

Open the failing run → the `Try to create a git tag` step → copy the **last ~20 lines**. The signal you want:

```text
fatal: unable to access 'https://github.com/…':
  The requested URL returned error: 403
Error: Resource not accessible by integration
  GITHUB_TOKEN permissions for 'contents' are insufficient
```

Open visualization **17.2** and compare your log with the three scenarios — see how the *pattern* is what Copilot keys on.

## Step 4 · Build the prompt

Open visualization **17.3** and toggle the prompt parts — note what each contributes. Then, in **Copilot Chat**, assemble the prompt against *your* log:

```
[fail log tail from Step 3]

Repo: <your-owner>/ci-demo · the job tries to create a git tag via gh api.
Constraint: the workflow currently declares `permissions: contents: read`.

1) Diagnose what failed and why.
2) State the root cause.
3) Propose a minimal YAML fix as a diff.
4) Tell me how to verify.
```

> **Why this works (guide §3):** context is the whole game. Log + workflow file + repo intent = a fix Copilot can be held accountable for. Copilot Chat can open the workflow file from the workspace — point it at `.github/workflows/troubleshoot-demo.yml` so it reads the real `permissions:` block.

## Step 5 · Diagnosis first — review before you trust

Open visualization **17.2** to see what a *good* Copilot answer looks like. Compare with yours. A strong answer:

1. Names the **root cause** (the `403` from a token that lacks `contents: write` — Module 15 least-privilege).
2. Proposes the **minimal fix** — either add `contents: write` to the job, or (better) ask whether you should even create tags from CI at all.
3. Tells you **how to verify** (re-run, check the tag appears).

> **The discipline (guide §4):** ask for the diagnosis *before* the fix. A fix without a diagnosis is a guess. If Copilot's explanation feels generic, check the environment before trusting the workflow change.

## Step 6 · The Module 15 guardrail pass

Open visualization **17.4** and click each guardrail. Run them against Copilot's proposed fix:

1. **Permissions unchanged?** — did Copilot widen `permissions:` (it probably needs to, *to the job only*)?
2. **SHA pins intact?** — no new tag-pinned `uses:`.
3. **No `pull_request_target`?** — reject if introduced.
4. **Real root cause?** — the fix addresses the 403, not a workaround.

Apply the fix **on the job that needs it** (not workflow-wide), push, re-run, and confirm the tag was created:

```bash
gh api repos/${{ github.repository }}/git/refs/tags/demo-v1 --jq '.ref'
```

## Step 7 · Bonus — troubleshoot the OIDC failure class

If Setup 12 is complete and you have a Lab 11-style ACR push workflow, intentionally drop `id-token: write` from the push job and re-run. Feed that log tail to Copilot (visualization **17.2 · Registry auth failure**) and compare its diagnosis with Lab 11's troubleshooting row.

---

## Expected outcome

- A failing run, a copied log tail, and a Copilot diagnosis that names the true root cause.
- A **PR-ready diff** you reviewed against the Module 15 guardrails and merged.
- The fix verified by a green re-run and the tag existing in the repo.

## Key takeaways

- **Context beats prompting tricks** — attach the workflow file and the failing job.
- **Diagnosis first, fix second** — a fix without a cause is a guess.
- **Never paste secrets into the prompt** — the chat transcript is a leak surface (Module 15).

## Troubleshooting

| Symptom | Fix |
|---|---|
| Copilot's diagnosis is generic/confident-but-wrong | Give more context (workflow file, repo, runner type) or check the environment — a thin prompt gets a guess |
| Copilot proposes a big rewrite | Ask for "a minimal diff that only fixes the root cause" (17.3 formula) |
| Fix proposes `pull_request_target` | Reject — it runs PR code with write privileges (Module 15 §2) |
| Re-run still fails after fix | Re-read the tail: is it the *same* error, or a *new* one deeper in the pipeline? Iterate diagnosis → fix → verify |
| You pasted a masked secret into the prompt | Rotate the secret immediately (Module 15); treat the transcript as exposed |

## What's next

Copilot turns diagnosis into fixes fast. **Lab 18** applies the authoring craft Copilot was built for: dynamic `discover → fromJSON` matrices and API-driven decisions.