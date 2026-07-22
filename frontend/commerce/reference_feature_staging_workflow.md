---
name: feature/staging is the shared deploy branch — merges here deploy to dev env
description: How the team uses feature/staging on ms-backoffice-service and frontend-commerce, and the clean recipe when it diverges during a session (reset local + re-merge from origin feature branch)
type: reference
---

`feature/staging` on both `ms-backoffice-service` and `frontend-commerce` is the **shared deploy branch for the dev environment**. When a feature is ready, it gets merged into `feature/staging`; the dev-env deployment picks up from there. Israel confirmed this convention on 2026-07-02.

## Rules

- **Multiple team members push to `feature/staging` in parallel.** Divergence during a session is normal, not a red flag.
- **Never force-push** `feature/staging`.
- **Never rebase** `feature/staging`.
- Feature work is done on separate branches (e.g., `feature/survey-admin-crud`) — those get pushed to origin BEFORE being merged into `feature/staging`. This is the invariant that makes the "reset local + re-merge" recipe below safe.

## Symptom: same features appear as different merge commits on local vs remote

Two ways the same feature can land on `feature/staging`:

- **GitHub PR merge** — the PR merge produces a commit like `Merge pull request #38 from corepark/feature/sms-usage`.
- **Direct local merge** — `git merge feature/sms-usage --no-ff` on someone's machine produces `Merge feature/sms-usage into feature/staging`.

These are the same content but different commit hashes. If two devs did each independently, the local and remote histories diverge with duplicate content.

## Recipe when your local `feature/staging` diverges from origin

Precondition: your feature branch (say `feature/survey-admin-crud`) is already pushed to origin. If not, push it first (`git push -u origin feature/survey-admin-crud`).

```bash
# 1. Fetch to see what's new
git fetch origin feature/staging

# 2. Reset local feature/staging to origin — drops your local merge + any duplicate merges
git reset --hard origin/feature/staging

# 3. Re-merge your feature branch from its origin tip (stable and pushed)
git merge origin/<your-feature-branch> --no-ff -m "Merge <your-feature-branch> into feature/staging"

# 4. Resolve any conflicts (usually the file both sides touched — nav / routes / lock)

# 5. Optionally reconcile the lock file
pnpm install

# 6. Push
git push origin feature/staging
```

**Why this works.** Step 2 discards your local `feature/staging` state including any stale merge you may have done earlier. Step 3 re-merges your feature branch's origin tip (stable, pushed, reviewable) against the freshest `feature/staging`. The resulting merge commit is clean — no duplicate merges, no stale conflicts.

**When to NOT use it.** If your local `feature/staging` has commits that DON'T live on any other branch — direct commits made to staging with no feature branch counterpart. That's rare on this team (features live on their own branches). If it happens, cherry-pick those commits into a real feature branch first, push, then use the recipe.

## Applied on 2026-07-02

FE `feature/staging` diverged during the session (10+ parallel merges landed on remote: sms-usage, collapsible-sidebar-menu, stayntouch, valet-attendant, English-convention docs commit). Applied the recipe — reset + re-merge from `origin/feature/survey-admin-crud`. Conflict-free this time because remote's navigation refactor and the survey branch already used the same `menuGroups` data model.
