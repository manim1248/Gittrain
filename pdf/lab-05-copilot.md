# Lab 05 · AI-Assisted Workflow Design with GitHub Copilot

> **Confidential · Stalwart Learning**
> Module 05 · Guided lab · Session 1
> Companion: `guides/module-05-ai-assisted-workflow-design-copilot.md` · Visualization: `module-05-visualization.html`

| | |
|---|---|
| **Objective** | Use Copilot Chat + inline suggestions to **scaffold**, **explain**, **research**, and **refactor** workflows — then run the guardrail checklist before anything merges |
| **Time** | ~55 min (guided) |
| **Prerequisites** | Setup 4 complete (Copilot enabled in VS Code), Labs 01–04 complete |
| **Files you create** | `.github/workflows/copilot-ci.yml`, `actions/build-node/action.yml` (from Lab 04, refactored) |

> This lab *is* the guardrail lesson. Copilot drafts are starting points, not review shortcuts.

---

## Pre-flight (2 min)

- [ ] Copilot + Copilot Chat icons active in VS Code status bar (Setup 4)
- [ ] `ci-demo` open in VS Code with `pipeline.yml` from Lab 03

## Step 1 · Scaffold a workflow from a description

1. Open **Copilot Chat** (⇧⌘I).
2. Paste a *specific* prompt — vague prompts produce vague YAML:

> Create a GitHub Actions workflow for a Node 20 service. Trigger on push to main and on pull_request. Jobs: lint and test (parallel), then build that waits for both. Use actions/checkout@v4, actions/setup-node@v4, actions/upload-artifact@v4. Steps: npm ci, npm run lint, npm test. Upload test results as an artifact. Pin third-party actions by commit SHA with the version tag in a comment, and set least-privilege permissions with contents: read.

3. Read the draft **before** applying it. Ask follow-ups if it's wrong: *"make lint and test run in parallel with needs: lint"*, *"add an explicit permissions block"*.
4. Save as `.github/workflows/copilot-ci.yml` and push — but **do not merge yet**. Step 5 is the gate.

## Step 2 · Explain an unfamiliar workflow or action

Pick an action you haven't used. Ask Copilot Chat:

> What does `actions/upload-artifact@v4` do? What inputs does it take, what permissions does it need, and is it from a verified publisher? What would I pin it to?

Then ask it to explain the workflow you built in Lab 01:

> Explain this workflow job by job. What runs, on what, and why? Flag anything insecure or risky.

Use the explanation to double-check the scaffold from Step 1.

## Step 3 · Research before you adopt

Ask Copilot to **recommend options** (it's a research aid, not an authority — verify in the Marketplace):

> List 3 well-known third-party actions for publishing Docker images to GHCR, with their refs and whether they're from verified publishers. Which would you recommend for a production pipeline and why?

Compare Copilot's answer against the Marketplace (`github.com/marketplace`) before relying on it.

## Step 4 · Refactor with Copilot — extract a composite action

Open `.github/workflows/pipeline.yml` from Lab 03 and ask Copilot Chat:

> In this workflow, the checkout + setup-node + install steps repeat in every job. Extract them into a composite action at actions/build-node/action.yml and replace the repeated steps with a single uses step. Preserve inputs and behavior exactly. Add shell: to every run: step.

Review the diff carefully — this is where Copilot most often sneaks in behavior changes:
- ✔️ same inputs, same action refs, same order of operations
- ⚠️ every composite `run:` has an explicit `shell:`
- ⚠️ any workflow-level `env:` that the composite can't inherit has been converted to an input

Push the refactor and run the workflow — behavior must be identical to before.

## Step 5 · The guardrails — before anything merges

Run the full checklist on everything Copilot generated (the visualization `module-05-visualization.html` §5.3 has this as an interactive gate):

1. **Human read-through** — would you have written each step yourself?
2. **Syntax/schema** — VS Code Actions extension shows no red squiggles; workflow editor on github.com agrees.
3. **actionlint** — the sneaky-errors linter. Run it locally:
   ```bash
   # install once
   brew install actionlint          # macOS; or curl script from rhysd/actionlint
   actionlint .github/workflows/copilot-ci.yml
   ```
4. **Security pass** — no floating refs (`@main`, `@latest`); explicit least-privilege `permissions:` (Copilot often omits it entirely); no secrets echoed to logs.
5. **Watch it run** — push to a branch, open a PR, confirm the run behaves as expected.
6. **Test the failure path** — break the test step once and confirm `needs`/`if:` behave as you assumed, then revert.

When all six are green, merge. Then complete the interactive gate in the visualization for your own records.

## Step 6 · Debug with Copilot (preview of Session 4)

1. Deliberately break a step: `- run: exit 1` in a test job. Push.
2. Copy the **failed step's log** from the Actions page.
3. Paste into Copilot Chat:

> Here is a failing GitHub Actions run log: <paste>. What is the root cause and what should the fix be? Propose the corrected YAML as a diff, explaining each change.

4. Apply the fix, re-run green, and note how the guardrails in Step 5 apply to Copilot's *fix* too.

---

## Expected outcome

- A Copilot-scaffolded workflow that survived the **6-step guardrail checklist** and merged.
- One Copilot-driven **refactor** (composite extraction) with identical behavior.
- One Copilot-powered **debugging round** on a real failure.

## Key takeaways

- **Prompt specificity drives output quality** — events, toolchain, steps, constraints, and your conventions.
- **Copilot accelerates, it doesn't review.** Every artifact passes: read → syntax → actionlint → security → live run → failure path.
- The same describe→generate→review loop transfers to ADO pipelines you're migrating (guide §7).

## Troubleshooting

| Symptom | Fix |
|---|---|
| Copilot suggests nothing | Re-run **GitHub Copilot: Sign in**; check status bar icon; restart VS Code (Setup 4) |
| Draft is valid YAML but references unknown actions | Ask Copilot to verify refs against the Marketplace; use `actions/*` names you can check |
| Composite refactor changed behavior | The top `env:` was silently dropped — convert it to inputs in the composite; confirm `shell:` on every `run:` |
| actionlint not installed / blocked by proxy | `brew install actionlint` (macOS) or `go install github.com/rhysd/actionlint/cmd/actionlint@latest`; the curl installer needs `github.com` access |