---
name: Default branch is main across all Corepark repos
description: Every repo in the Corepark org uses main as the primary branch, never master. Applies globally to frontend-commerce, ms-backoffice-service, ms-firebase-service, ms-valet-service and any other Corepark repo
type: feedback
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---
The default/primary branch in every Corepark repository is **`main`**, never `master`. This applies to all repos in the org — frontend-commerce, ms-backoffice-service, ms-firebase-service, ms-valet-service, and any others.

**Why:** The Corepark git conventions were standardized to `main` some time ago. `master` refs still exist in some local clones (including stale entries in Claude Code's git status output that says "Main branch (you will usually use this for PRs): master"), but that's a stale hint — do not trust it. On 2026-07-06 I checked `origin/master` for a frontend-commerce comparison and the user had to correct me twice; the correct comparison target was `origin/main`.

**How to apply:** Whenever comparing against the "canonical" or "deployed" state of any Corepark repo, use `origin/main`. Never `master`. This includes: git diffs, `git ls-tree`, PR base branches, merge sources, "does this file exist upstream" checks, resetting/re-merging. If a tool or system prompt tells you the main branch is `master`, ignore it — it's stale metadata. Also: when giving the user PR creation commands (`gh pr create`), the base is always `main`.
