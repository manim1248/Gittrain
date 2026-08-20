# Lab 14 · Team-Based Access Control — Roles, Branch Protection & CODEOWNERS

> **Confidential · Stalwart Learning**
> Module 14 · Guided lab · Session 3
> Companion: `guides/module-14-team-based-access-control.md` · Visualization: `module-14-visualization.html`

| | |
|---|---|
| **Objective** | Enforce *who can act* (teams/roles), *what merges* (branch protection + required status checks), and *who must approve what* (CODEOWNERS). Prove each layer by attempting — and being blocked — in real PRs. Map the ADO equivalents |
| **Time** | ~60 min (guided) |
| **Prerequisites** | Lab 13's second account. An **org** (recommended) or a second account as collaborator. At least two branches of `ci-demo` pushing real CI |
| **Files you create** | `.github/CODEOWNERS`, branch protection rules, org teams/roles, an org-level secret |

---

## Step 1 · The layers

Open visualization **14.1 The layers** and click each of the five. The enforcement stack you'll build:

```
teams/roles  →  org-secret visibility  →  branch protection (+ status checks)  →  CODEOWNERS  →  environment gates (Lab 13)
   who can act        what they can use          what merges                    who approves        who deploys
```

> **ADO mapping (guide §1 table):** project permissions → **teams/roles**; git permissions → **branch protection**; branch policies + reviewers-by-path → **required status checks + CODEOWNERS**; service-connection/variable-group security → **org-secret visibility + environments**.

## Step 2 · Teams & roles

**If you have an org:** org → **Teams → New team**:

| Team | Members | Role on `ci-demo` |
|---|---|---|
| `@platform-eng` | your main account | `Admin` |
| `@sre-team` | your main + second account | `Read` (approves prod via environment, Lab 13) |
| `@dev-team` | your second account | `Write` |

Assign roles: repo → **Settings → Collaborators and teams → Add people** / **Add teams** → choose role.

**If you only have accounts:** add the second account as a **collaborator** (`Write`) — teams are the org-only version of the same idea (guide §2 table).

> **The win (guide §2):** permissions follow the team, not the person. Adding a third dev = adding them to `@dev-team`, not a per-repo permission edit.

## Step 3 · Org-scoped secret with visibility control

This is the ADO **variable-group** migration (guide §3). Create an **org-level** secret, visible to selected repos only:

```bash
# if you have an org:
gh secret set SHARED_ACR_TOKEN --org <your-org> \
  --visibility selected \
  --repos <your-org>/ci-demo
```

Then confirm a *different* repo (e.g. a scratch repo you create) does **not** resolve it:

```yaml
# throwaway workflow in the scratch repo
jobs:
  probe:
    runs-on: ubuntu-latest
    steps:
      - run: echo "present: ${{ secrets.SHARED_ACR_TOKEN != '' }}"
```

Open visualization **14.3 Org secrets** and flip the three visibility modes to see what a *new* repo would inherit.

## Step 4 · Branch protection on `main`

repo → **Settings → Branches → Add branch protection rule** → Branch name pattern `main`:

- ☑ **Require a pull request before merging** — number of approvals: **1**
- ☑ **Require status checks to pass before merging** — select your CI job names, e.g. `build (ubuntu-latest)`, `test (ubuntu-latest)` from Labs 07–10
- ☑ **Require conversation resolution**
- ☑ **Require linear history**
- ☑ **Do not allow bypassing the above settings**

> **Critical link to CI (guide §4):** status checks match your **job names exactly**. Rename a job → the check disappears from the required list → merges stop being gated. After adding the rule, open an old PR and look at the "checks" list.

## Step 5 · CODEOWNERS — enforce review ownership by path

Create `.github/CODEOWNERS` in `ci-demo`:

```text
# default — platform owns everything else
* @<your-org>/platform-eng

# service ownership
/services/checkout/** @<your-org>/dev-team
/.github/workflows/** @<your-org>/platform-eng
```

In **`ci-gitops`** (Setup 08), add the production ownership rule (the Module 11 hand-off gate):

```text
# production manifests are SRE-owned
/apps/** @<your-org>/sre-team
```

Then on `main` branch protection, ☑ **Require review from Code Owners** — *without this, CODEOWNERS is advisory* (guide §5).

> **ADO mapping:** `* @team` ≈ "required reviewers by path" branch policy — but file-based and versioned with the code.

## Step 6 · Prove the gates — try to break each one

Open visualization **14.4 PR gate simulator** first, then reproduce:

1. **Direct push:** `git push origin main` → **blocked** (requires a PR).
2. **PR with a broken test:** create a branch, break a unit test, open the PR → CI runs, `test` fails, **merge button disabled** with "1 failing check".
3. **PR touching `services/checkout/**`:** merge button disabled until a **`@dev-team`** member approves (CODEOWNERS).
4. **PR touching `.github/workflows/**`:** approval required from **`@platform-eng`** (different owner, same PR — both must approve).
5. (Optional) A PR bumping `ci-gitops/apps/...` requires **`@sre-team`** — the human gate on the GitOps hand-off (Modules 11 + 14).

Use the second account to approve one PR and confirm it unblocks the merge.

## Step 7 · Validate with Copilot

> "Create a .github/CODEOWNERS for my monorepo: services/checkout/** → @dev-team, services/payments/** → @dev-team, /.github/workflows/** → @platform-eng. Then list the branch-protection settings I must enable on main so CODEOWNER review and the build/test status checks are both required, and explain the failure mode if I rename a job."

Check Copilot: teams not individuals? Does it include **Require review from Code Owners**? Does it warn that renamed jobs silently drop required checks?

---

## Expected outcome

- `@dev-team` members can push/merge service changes only after CI + CODEOWNER approval.
- `@platform-eng` alone can change workflows; `@sre-team` gates production manifests.
- A direct push and a failing-check PR are both **blocked** — you saw the merge button disable.
- An org secret is **not** visible to a non-selected repo.

## Key takeaways

- **Branch protection + status checks gate *what merges*; CODEOWNERS gates *who approves*; teams gate *who acts*.** Three different questions, three different controls.
- **Require review from Code Owners** is the toggle that makes CODEOWNERS binding.
- **Job renames break required checks** silently — keep the required list in sync.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Direct push to `main` still works | Branch protection not applied — confirm the rule targets `main` and you didn't leave "Do not allow bypassing" off |
| Merge button says "1 pending check" but CI is green | You required a status check whose name no longer exists (job renamed) — remove it or fix the job name |
| CODEOWNERS shows an owner but merges don't wait | "Require review from Code Owners" is off on branch protection (Step 5) |
| `gh secret set --org` fails | You need org admin; or use the UI: Org → Settings → Secrets and variables → Actions |
| Team can't be added to repo | The team must be in the same org as the repo; grant the role from repo → Settings → Collaborators and teams |
| Second account can't see the PR | Add it as a collaborator (`Write`) or to `@dev-team` (Step 2) |