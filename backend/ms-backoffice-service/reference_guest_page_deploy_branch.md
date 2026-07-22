---
name: frontend-guest-page deploys from `develop`, not `feature/staging`
description: The odd one out — every other Corepark repo uses `feature/staging` as the shared dev deploy branch, but frontend-guest-page uses `develop`
type: reference
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
# frontend-guest-page deploy branch

**Rule:** in `frontend-guest-page`, the shared dev deploy branch is `develop`, **not** `feature/staging`.

Every other Corepark repo in this ecosystem (`ms-valet-service`, `ms-gateway-service`, `ms-backoffice-service`, `frontend-commerce`, `frontend-backoffice`, …) uses `feature/staging` as the shared branch that deploys to the dev environment. `frontend-guest-page` is the odd one out — it still uses the older `develop` convention.

## When it matters

- When you merge a feature into "staging" for `frontend-guest-page`, target `develop`.
- `frontend-guest-page` has no `feature/staging` branch on the remote — `git fetch origin feature/staging` will fail with `couldn't find remote ref feature/staging`.

## Related

- `reference_feature_staging_workflow.md` — the general `feature/staging` recipe for the other repos.
- `project_vehicle_info_ecosystem.md` — the feature that surfaced this on 2026-07-17.
