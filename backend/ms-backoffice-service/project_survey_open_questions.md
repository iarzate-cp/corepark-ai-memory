---
name: Survey CRUD — design questions with resolutions
description: The six design decisions raised at kickoff, and how each was resolved on 2026-07-01
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
**Status:** all six blocking questions were answered on 2026-07-01. Recorded here for future sessions so we don't relitigate.

| Question | Decision |
|---|---|
| Multiple active surveys per (operator, location)? | **One active per pair.** Enforced via `existsActiveSurveyForLocation` check → `ERR_SURVEY_005` on create. |
| Editing survey that has answers — bloqueo / versionado / in-place? | **Versionado automático:** editar = create new + deactivate old. Not yet implemented (pending PATCH endpoint). |
| `surveyTypeId` / `questionTypeId` — user-selectable or fixed? | **Fixed enum for MVP.** No catalog endpoint. Only `surveyTypeId=1` and `questionTypeId ∈ {1: Rate, 2: RadioButton}` observed. |
| Language catalog — user picks from `custom.cat_language`? | **Yes, configurable per survey.** Default at create = EN+ES+FR (1, 2, 5). Front will expose selector in Slice 3. |
| Reorder questions/options? | **Not in MVP.** Requires a `position` column that doesn't exist in schema today. Insert order = display order. Add later if needed. |
| Delete semantics — soft or hard? | **Soft delete** via `company.survey.deactivated_at`. Not yet implemented (pending DELETE endpoint). |

**Also decided:**
- **Preview URL for Elias/customer service:** implemented via preview petition mechanism (see `project_survey_preview_petition.md`).
- **Editor UX:** dedicated page (not dialog), tabs per language, pre-fill with the 2 default questions from the `.sql` provisioning script.

**How to apply:** treat these as settled unless the user reopens one. If a new question arises during implementation (e.g., "should editing block re-answers by clients?"), raise it explicitly rather than assuming.
