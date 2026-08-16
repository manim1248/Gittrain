# Setup 4 · GitHub Copilot for GitHub Actions

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration

## Objective

Enable GitHub Copilot on the account, connect it to VS Code, and verify it can see and help with workflows. **Required for Module 05 and used from Lab 01 onward as a working tool.**

> The course prerequisite is a **Copilot Business or Enterprise licence**. If your org hasn't purchased it, start the **30-day Copilot Business trial** (see Step 1b) before Session 1.

## 1. Get the licence

### 1a · Business (recommended for this course)

- **Org admin** flow: https://github.com/orgs/<org>/settings/copilot → **Enable Copilot for your organization** → choose *Business* → assign seats to yourself (and teammates).
- **Or start a trial**: https://github.com/settings/billing/summary → **Billing and plans** → *Copilot* → **Start a Business trial** (30 days, no credit card for trial). As an individual you can also start a Pro trial from your own Copilot settings.

### 1b · Individual fallback (if Business/Enterprise is unavailable)

- **Copilot Free** is available for individual accounts: https://github.com/settings/copilot → **Get Copilot Free**. Includes ~2000 completions / 50 chat requests per month — enough to follow the Module 05 lab. Some org-level Copilot features (e.g. policies) won't apply, which is fine for the labs.

## 2. Enable it on your account

1. https://github.com/settings/copilot → confirm it shows **Enabled**.
2. (Business) org members may need to click **Accept** the Copilot invite from the org's notification bell.

## 3. Connect Copilot to VS Code

1. Extensions from Setup 3: **GitHub Copilot** and **GitHub Copilot Chat** installed.
2. **Cmd+Shift+P** → **"GitHub Copilot: Sign in"** → authorize in the browser.
3. Status check: the **Copilot icon in the status bar** (bottom-right) shows a filled circle when active (✕ / open circle means disabled).
4. Open **Copilot Chat** (⇧⌘I / Ctrl+Shift+I) — a panel opens with a text box.

## 4. Verify it works on workflows

Ask Copilot Chat a workflow question — it should answer from context:

> "What does `actions/upload-artifact@v4` do, and what inputs does it take?"

Then verify inline completion: in an empty `.github/workflows/test.yml`, start typing:

```yaml
name: verify
on: [push]
jobs:
```

and accept the suggestion with **Tab**.

## 5. (Enterprise only) Your admin may set policies

If your org enforces Copilot policy (e.g. suggestions filtering, chat in private code), confirm those don't block the training account. Most defaults are fine for the labs.

## Outcome

- Copilot licence active on the account (Business/Enterprise preferred, Free as fallback).
- VS Code signed in; Copilot + Chat icons active.
- Copilot answered a question about a workflow and produced an inline completion.

## Troubleshooting

| Problem | Fix |
|---|---|
| "Copilot is not enabled" in VS Code | Re-run **GitHub Copilot: Sign in**; check account at `github.com/settings/copilot`; org invite may be pending |
| Greyed-out / status ✕ | `Cmd+Shift+P` → "GitHub Copilot: Toggle Copilot Completions"; restart VS Code |
| Chat answers "no access" | Confirm the seat is assigned to **this** account, not a different one |
| Corporate proxy blocks Copilot | Allow `copilot-proxy.githubusercontent.com` and `api.githubcopilot.com` in the proxy |

> **Heads-up for the course:** Copilot is a *working tool from day one* — the Module 05 lab treats it as an authoring aid, and every earlier lab is written so you could (in principle) scaffold it with Copilot. The guardrails checklist in `module-05/lab-05-copilot.md` applies to everything Copilot generates.