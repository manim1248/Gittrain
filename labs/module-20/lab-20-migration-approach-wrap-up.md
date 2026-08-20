# Lab 20 · Migration Approach & Wrap-Up — Plan Your Own Move

> **Confidential · Stalwart Learning**
> Module 20 · Guided lab · Session 4
> Companion: `guides/module-20-migration-approach-and-wrap-up.md` · Visualization: `module-20-visualization.html`

| | |
|---|---|
| **Objective** | Run the 7-phase migration checklist against **your real Azure DevOps estate**, produce a cutover plan, and close the course with open Q&A. This lab produces a *document you'll actually use*, not just a practice artifact |
| **Time** | ~50 min (guided) + open Q&A |
| **Prerequisites** | None beyond the course. Bring 2–3 real ADO pipelines you own or will migrate. Copilot (Setup 04) recommended for Step 6 |
| **Files you create** | `migration-plan.md` at the repo root of `ci-demo` (or your docs location) |

---

## Step 1 · The 7 phases

Open visualization **20.1** and click through all seven phases — inventory, concept mapping, target-state design, lift & shift, harden, optimise, cutover. Create `migration-plan.md` with a phase header for each:

```markdown
# Migration Plan — Azure DevOps → GitHub Actions

## Phase 1 · Inventory
| Pipeline | Owner | Risk (build/CD) | Touches prod? | Modules |
|---|---|---|---|---|
| (your pipeline) | (team) | build | no | 01–10 |
| ... | ... | ... | ... | ... |

## Phase 2 · Concept mapping
## Phase 3 · Target state
## Phase 4 · Lift & shift (1:1)
## Phase 5 · Harden
## Phase 6 · Optimise
## Phase 7 · Cutover
```

## Step 2 · Phase 1 — inventory your real estate

List **2–3 real ADO pipelines** you own. For each capture: name, stages, key tasks, variable groups, service connections, approvals, and whether it can deploy to production. Classify **risk** (build-only vs deploy-capable) and **module coverage** (which of this course's modules map to it).

> **Guide §2:** the inventory is the input to everything else — a pipeline you forget here will surprise you at cutover.

## Step 3 · Phase 2 — concept-mapping your pipelines

Open visualization **20.2** and click rows of the ADO→GitHub map. For each pipeline in your inventory, note the Actions equivalents using the table. Call out the two concepts with **no ADO equivalent** — they're where the design work goes:

1. **Least-privilege `permissions:`** (Module 15) — ADO's token defaulted to broad scope.
2. **Reusable workflows / composite actions** (Module 10) — your YAML templates, but better.

## Step 4 · Phase 3 & 4 — design target state, then lift & shift

Open visualization **20.3** and answer the five decision points **for your estate**:

| Decision | Your answer | Consequence |
|---|---|---|
| Same build shape across services? | reusable workflow vs per-service (10) | template library |
| Cloud credentials in use? | OIDC adoption (12) | removes service connections |
| Who approves production? | reviewers + branches (13) | gate ownership |
| Monorepo or many repos? | matrix vs path filters (07/18) | repo topology |
| Self-hosted runners? | GitHub-hosted vs self-hosted (03) | runner estate |

Then pick **one pipeline** and sketch its 1:1 ported workflow — task-by-task, parity first (Phase 4). Use Copilot if helpful, but keep it a *1:1 port*; optimisation comes in Phase 6.

## Step 5 · Phase 5 & 6 — harden and standardise

Open visualization **19.3** (the hardening checklist) and walk it against your sketch:

- Least-privilege `permissions:` floor + per-job scopes
- SHA-pinned `uses:` + Dependabot
- No secrets in logs; OIDC everywhere
- No `pull_request_target`; secret scanning on

For Phase 6, note where **caching** (08), **matrices** (07), and **dynamic generation** (18) would replace one-off ADO task chains — but keep it as *future work* on the plan, not part of the parity port.

## Step 6 · Phase 7 — the cutover plan

Open visualization **20.4** and click the three cutover phases. Write your own timeline into the plan:

- **Weeks 1–2:** port 1–2 low-risk pipelines; run ADO + Actions **side-by-side** on the same commits; diff the results.
- **Weeks 2–4:** move branch-protection status checks (14) so Actions gates merges.
- **Week 4+:** cut over deploy-capable pipelines and environment gates (13); decommission ADO behind a **documented runbook**.

Capture the **rollback path** explicitly: *"revert the branch-protection rule; ADO becomes the gate again"*.

Then ask Copilot to sanity-check the plan:

> "Act as a migration consultant. Review my migration-plan.md: is the phase order sound? Flag anything that skips the hardening pass, keeps long-lived cloud credentials, ignores the GitOps hand-off boundary (Module 11), or proposes a big-bang cutover without a rollback path."

## Step 7 · Course wrap-up — the one-diagram view

Open visualization **20.5** to see the whole course in one diagram. Check your confidence on the course's four learning objectives:

1. Author, secure, and operate GitHub Actions workflows for containerised microservices.
2. Apply team-based access control (environments, branch protection, CODEOWNERS).
3. Use GitHub Copilot to accelerate workflow design and debugging.

Then open Q&A: bring the pipelines you just inventoried. The instructor maps each against this checklist live.

---

## Expected outcome

- A `migration-plan.md` you can take into your team: inventory, concept map, decisions, a 1:1 port sketch, a hardening list, and a dated cutover with a rollback path.
- You can answer: *"What stays in ADO, what moves first, what does the safety net look like?"*

## Key takeaways

- **Migrate repo-by-repo, side-by-side** — parity before cutover; big-bang is how migrations fail.
- **Copy the intent, not the YAML** — the mapping is conceptual; `permissions:` and reusable workflows have no ADO equivalent.
- **OIDC is the migration's highest-value win** — every service connection you federate is a long-lived credential you no longer store.

## Troubleshooting

| Symptom | Fix |
|---|---|
| "We have too many pipelines to inventory" | Inventory by *risk tier* first (deploy-capable, prod-touching); build-only pipelines can move in bulk |
| Can't decide reusable vs per-service | Default to reusable workflow (10) for the common shape; use generation (18) only when repos diverge a lot |
| Team wants everything on Actions by Friday | That's the big-bang pitfall — insist on side-by-side parity runs (Phase 7) and a rollback rule |
| Copilot's plan adds out-of-scope modules | Remind it of the boundaries: no Flux deep-dive (11), no blue-green/canary (Flux/Flagger), no Terraform/IaC |
| Prod approvals feel too slow | Right-size the gates (13): friction at staging/prod, none at dev — approval fatigue kills pipelines |

## What's next

The course is complete. The last thing this module leaves you is the ownership question: who approves environments, who owns action upgrades, and who keeps the GitOps boundary clean (11). That's your post-course operating model.