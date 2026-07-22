---
name: Survey Wave 2 deploy workflow — BE + FE coordinated
description: The concrete sequence used on 2026-07-07 to deploy a breaking envelope change to dev without a windowed contract mismatch. Reusable recipe for future breaking-shape refactors
type: project
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
Wave 2 (envelope migration `Response` → `ApiResponse<T>`) was a **breaking contract change**: JSON shape changes from `{ code, ...customFields }` to `{ code, data, message, errors }`. Backend and frontend must land in dev at effectively the same time or the UI breaks for the window between.

## The sequence used on 2026-07-07

1. **Local dev + push feature branches:**
   - `ms-backoffice-service` `feature/survey-admin-crud` pushed (`0b7b389..f381453`) with Phase 1 + Phase 2 commits.
   - `frontend-commerce` `feature/survey-admin-crud` pushed (`2bc30bb..5fb9350`) with the consumer refactor.

2. **Manual end-to-end test in local first.** Israel spun up both `ms-backoffice-service` and `frontend-commerce ng serve` and exercised all 7 admin endpoints from `/settings/survey`. Confirmed the JSON shape matches on both sides *before* touching staging.

3. **Fetch `feature/staging` on both repos** to verify no divergence with origin:
   ```
   git fetch origin feature/staging
   git log --oneline HEAD..origin/feature/staging     # should be empty
   ```

4. **Merge BE first, then FE — both with `--no-ff`** and descriptive merge commit messages that reference the paired commit on the other repo:
   ```
   git checkout feature/staging
   git merge --no-ff feature/survey-admin-crud -m "..."
   git push origin feature/staging
   ```
   BE went first: `0543836`. Then FE: `87a2d8c`. Total elapsed between the two `git push` commands: a few minutes.

5. **Verified dev end-to-end** once both pipelines finished.

## Why `--no-ff` and not a fast-forward

- Preserves the merge point in `feature/staging`'s history, so `git log --graph` shows exactly when the pair went in.
- Allows targeted revert via `git revert -m 1 <merge-commit>` if the deploy explodes — reverting just the merge commit undoes the entire wave without touching sibling work.

## Conflict handling on the BE merge

- `ErrorCode.java` conflicted because both the firebase-cleanup slice (already in staging via `f841a03`) and the survey slice added entries. Add-only from both sides; resolution was to concatenate both blocks under labeled sections. No behavior conflict.
- Nothing else conflicted — the file moves were seen by git as renames because the packages moved atomically.

## What NOT to do (specific to this environment)

- Do NOT push BE and let it sit while FE is in review. The window between BE arriving in staging and FE arriving is exactly when `frontend-commerce` in dev shows a broken `/settings/survey` (it expects `r.locations` / `r.surveys` / `r.survey` but BE now returns `r.data`).
- Do NOT rely on the FE being tolerant to shape changes. Angular services strongly type responses; a shape mismatch produces `undefined` where a list was expected and the UI shows empty state.
- Do NOT merge FE first. If FE lands first, dev sees a working UI that talks to the still-legacy BE — but as soon as BE lands, the FE breaks because it was already reading `.data`. Same window, just inverted.

## The safe order in general

Backend push → backend merge to staging → frontend push → frontend merge to staging. If FE takes longer to deploy than BE (Angular builds are chunky), you can invert the merges to give FE a small head start, but only if the FE is written to tolerate the transition (e.g. defensive `r.data ?? r.locations`, which we did NOT do here). We used the simpler "back-to-back merges" strategy — a few minutes of exposure is acceptable in dev.

## Reusability

This recipe is the default for any future breaking shape change on the API contract. The next likely case is migrating `SurveyController` (consumer flow) — same drill, but coordinated with `frontend-survey` (a production client, so the exposure window has to be zero or near-zero).
