# Setup 1 · GitHub Account (and optional Organization)

> **Confidential · Stalwart Learning** · GitHub Actions — CI/CD Enablement & Migration

## Objective

Create the GitHub identity you will use for all labs: a personal account for your own repo, and (recommended) an organization for org-level features used in Module 04 (shared reusable workflows, org secrets).

## 1. Create the personal account

1. Go to **https://github.com/signup**.
2. Enter an email, a password, and a username. Choose a professional username (it will appear in URLs like `github.com/<you>/ci-demo`).
3. Complete the puzzle verification, then **Verify email address** from the email GitHub sends.
4. Choose your plan: **Free** is enough for every lab in this course.
5. Confirm the account at `https://github.com` → profile.

> Corporate proxy note: if you see connection errors, add `github.com`, `api.github.com`, `*.githubusercontent.com`, and `*.actions.githubusercontent.com` to your proxy/firewall allow-list before continuing.

## 2. Harden the account (recommended)

- **Two-factor authentication**: GitHub Settings → **Password and authentication** → **Two-factor authentication** → enable (authenticator app recommended). Required for full API use on many orgs.
- Set your **default branch name**: Settings → **Repositories** → *Default branch* → `main`.
- Set a **profile name + email** (Settings → **Profile**) — GitHub uses these in your commit identity.

## 3. Create an organization (recommended)

Org-level features appear throughout the course: **org secrets**, **org variables**, and **reusable workflows shared across repos** (Module 04), and **team-based access control** (Session 3, Module 14).

1. Click **+** (top-right) → **New organization**.
2. Choose the **Free** plan (all labs fit; the "Team" plan is not required).
3. Name it, e.g. `yourcompany-actions-training`. Set *Personal account* as the owner.
4. Add your personal account (and your real team, if present) as members.
5. Verify the org exists at `https://github.com/<org>`.

> You can complete the labs with a **personal repo only**; the org becomes necessary for the org-scoped secrets in Module 04 step and Session 3.

## 4. Verify you can create repos

- Confirm **+ → New repository** works on your personal account.
- Confirm your org page shows **New repository** too (owner dropdown lists the org).

## Outcome

- Personal account with 2FA, default branch `main`.
- Optional org you own, ready for repo and secret creation.

## Troubleshooting

| Problem | Fix |
|---|---|
| Email not verified | Re-send from the GitHub verification email; check spam |
| "Username already taken" | Pick a suffix (e.g. `-devops`, `-training`) |
| Sign-up puzzle won't pass | Reload; it occasionally needs a refresh |