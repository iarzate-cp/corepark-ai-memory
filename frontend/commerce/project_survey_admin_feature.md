---
name: Survey admin CRUD — frontend shipped 2026-07-02
description: /settings/survey feature merged into feature/staging and pushed. Uses corepark-ui@0.0.23 with chevron-only cp-table toggle
type: project
---

**Snapshot as of end of 2026-07-02.** Feature shipped. `feature/staging` carries the merge; dev env picks up on next deploy.

## Branches on origin

- `feature/staging` — HEAD `d8b976e Merge feature/survey-admin-crud into feature/staging`. Includes the whole survey feature. Up to date with origin.
- `feature/survey-admin-crud` — HEAD `2bc30bb feat: use chevron-only toggle for survey table`. Available for PR reference. Up to date with origin.

## Commit lineage on `feature/survey-admin-crud`

    2bc30bb feat: use chevron-only toggle for survey table
    52e1142 feat: add /settings/survey admin CRUD
    51a6013 chore: drop local env config; bump corepark-ui to 0.0.22

## What lives at `/settings/survey`

Operator admin (Customer Service persona, e.g. Elias) manages per-location surveys with full CRUD.

### 5 dialogs + 2 supporting components + 1 listing page

All under `src/app/shared/components/*` with barrel `index.ts` re-exporting `FooComponent as Foo` + types. Import via `@components/<name>`.

| Component | Path | Purpose | Data shape | Result shape |
|---|---|---|---|---|
| `BasicSurveyDialog` | `basic-survey-dialog/` | 1-click provisioning | `{op, loc, name}` | `boolean` (created) |
| `CustomSurveyDialog` | `custom-survey-dialog/` | Full editor (dual-mode: create/edit) | `{op, loc, name, surveyUuid?}` | `boolean` (created or updated) |
| `SeeSurveyDialog` | `see-survey-dialog/` | Read-only view | `{surveyUuid, op, loc, name}` | `void` |
| `DeleteSurveyDialog` | `delete-survey-dialog/` | Preview + confirm + soft-delete | `{surveyUuid, op, loc, name}` | `boolean` (deleted) |
| `ReactivateSurveyDialog` | `reactivate-survey-dialog/` | Preview + swap notice + reactivate | `{surveyUuid, op, loc, name, hasActiveSurvey}` | `boolean` (reactivated) |

Supporting:
- `SurveyPreview` (`survey-preview/`) — reusable read-only body (tabs per language, welcome/completion, questions with options as chips). Consumed by Basic, See, Delete, Reactivate.
- `SurveyHistoryList` (`survey-history-list/`) — row-expansion content. Inputs `operatorCompanyId` + `parkingLocationId`. Auto-loads history via `queueMicrotask` in constructor. Renders Active/Archived badges + Reactivate button on archived. Output `reactivate: SurveySummary`.

### Listing (`pages/settings/survey/`)

- `SurveyComponent` — page component, uses `cp-table` with `[collapsible]="true"` + `[rowClickToggles]="false"` (chevron-only toggle) + `<ng-template cpExpand>` rendering `<survey-history-list>`.
- Columns: Location (sortable), Status, Questions, Languages, Created (`DatePipe` `'MMM d, y'`), Actions.
- Actions column consolidates row ops in `cp-menu` dropdown:
  - Row **with** survey → "Actions ▾" → See survey / Edit survey / Delete survey
  - Row **without** survey → "Add survey ▾" → Basic setup / Custom setup
- Row expansion (chevron cell only, not row body) shows history of all surveys for that location (active + archived).

## Navigation (post-merge state on `feature/staging`)

The sidebar is now organized with `menuGroups: readonly MenuGroup[]` (each group has label, icon, items) plus a small `rootLinks` for standalone items like Operators. Survey lives under the SETTINGS group at position last:

    {
      label: 'SETTINGS',
      icon: 'settings',
      items: [
        { label: 'Valet app', path: '/settings/valet-app' },
        { label: 'Guest page', path: '/settings/guest-page' },
        { label: 'Phone number', path: '/settings/phone-number' },
        { label: 'White label', path: '/settings/white-label' },
        { label: 'Screen config', path: '/settings/screen-config' },
        { label: 'Damage config', path: '/settings/damage-config' },
        { label: 'Survey', path: '/settings/survey' },
      ],
    }

## Data layer

### `@definitions/survey.d.ts`

- `SurveySummary`, `SurveyLocationStatus`, `SurveyDetail` + nested (`SurveyDetailMessage`, `SurveyDetailQuestion`, `SurveyDetailOption`, `SurveyDetailSentence`, `SurveyDetailSupportedLanguage`)
- Response wrappers: `SurveyLocationsStatusResponse`, `SurveyHistoryResponse`, `SurveyDetailResponse`, `CreateSurveyResponse`
- Editor draft types: `SurveyDraft`, `MessageDraft`, `QuestionDraft`, `OptionDraft`, `SentenceDraft`, `LanguageId`, `QuestionTypeId`

### `@services/surveys-service.ts` — 7 methods

All uuid-based methods send `Operator-Id` + `Location-Id` headers — backend enforces tenant isolation.

| Method | Signature | Verb |
|---|---|---|
| `getLocationsStatus` | `(opId) → Observable<SurveyLocationStatus[]>` | GET |
| `getLocationHistory` | `(opId, locId) → Observable<SurveySummary[]>` | GET |
| `getSurveyDetail` | `(uuid, opId, locId) → Observable<SurveyDetail>` | GET |
| `createSurvey` | `(opId, locId, draft) → Observable<CreateSurveyResponse['survey']>` | POST |
| `updateSurvey` | `(uuid, opId, locId, draft) → Observable<SurveyDetail>` | PATCH |
| `deleteSurvey` | `(uuid, opId, locId) → Observable<void>` | DELETE |
| `reactivateSurvey` | `(uuid, opId, locId) → Observable<SurveyDetail>` | POST |

### `@utils/survey-draft.ts` — domain helpers

- `LANGUAGES` catalog + `DEFAULT_LANGUAGE_ID`
- `buildBasicSurveyDraft()` — matches legacy template
- `surveyDetailToDraft(detail)` — wire → draft mapper
- Pure mutators: `addLanguage`, `removeLanguage`, `updateMessageField`, `addQuestion`, `removeQuestion`, `updateQuestionType`, `updateQuestionOptional`, `updateQuestionSentence`, `addOption`, `removeOption`, `updateOptionSentence`
- `isDraftValid(draft): boolean` — mirrors backend validation rules
- `prepareSubmitPayload(draft): SurveyDraft` — injects 5 synthetic options for Rate questions on submit

### `@utils/notify.ts`

- `notifyHttpError(notif, error, title)` — status-aware (5xx → 'error', 4xx → 'warning'), extracts server message

### `@utils/path-service-setter.ts`

```
SURVEYS: {
    LOCATIONS_STATUS: 'backoffice/v1/surveys/locations-status',
    HISTORY:          'backoffice/v1/surveys/history',
    CREATE:           'backoffice/v1/surveys',
    DETAIL:     (uuid) => `backoffice/v1/surveys/${uuid}`,
    UPDATE:     (uuid) => `backoffice/v1/surveys/${uuid}`,
    DELETE:     (uuid) => `backoffice/v1/surveys/${uuid}`,
    REACTIVATE: (uuid) => `backoffice/v1/surveys/${uuid}/reactivate`,
}
```

## App-wide setup

- `app.config.ts` has `provideNotifications({ position: 'bottom-right' })` — the whole survey flow uses `NotificationService` from `@corepark/corepark-ui`, NOT `MatSnackBar`.
- Route: `/settings/survey` → `pages/settings/survey/index.ts` (lazy).

## Design system dependency

`@corepark/corepark-ui@0.0.23` — components used in the survey feature:

- `DialogService` + `cp-dialog-content` (all 5 dialogs)
- `cp-table` + `CpCellDef` + `CpExpandDef` + `[rowClickToggles]="false"` (chevron-only toggle, listing)
- `cp-badge` (status, question type chips, active/archived history)
- `cp-menu` + `cp-menu-item` + `cpMenuTrigger` (Actions dropdowns)
- `cp-checkbox` (language toggles, confirm checkboxes)
- `cp-form-field` + `[cpInput]` (all editor inputs/textareas)
- `cp-select` (question type in Custom editor)
- `cp-switch` (Optional flag on questions)
- `cp-tab-group` + `cp-tab` (per-language tabs)
- `[cpButton]` (all buttons)
- `NotificationService` (all feedback)
- `DatePipe` (created column, history dates)

## Backend contract

- All uuid-based endpoints filter by `(operator, location, uuid)` on the backend. Wrong-tenant UUID returns `ERR_SURVEY_001` — the frontend renders it via `notifyHttpError` as a warning toast.
- `ERR_SURVEY_005` (409) fires from either the app-level check or the DB unique index (`uq_survey_active_survey`) — behaviorally identical from the FE perspective.
- Validation error codes (`ERR_SURVEY_006..011`) are all mirrored in `isDraftValid` — should never fire from the UI in normal use.
- `ERR_SURVEY_012` — FK / integrity violation. Only reachable via API tools; the UI only exposes valid IDs.

## Deploy pipeline

`feature/staging` on both `frontend-commerce` and `ms-backoffice-service` is the shared deploy branch. Merges here trigger dev-env deploys. See `reference_feature_staging_workflow.md` for the merge recipe when staging has diverged.

## Related docs (in ~/Documents/)

- `frontend-commerce-survey-qa-2026-07-02.html` — technical QA plan
- `survey-admin-operator-guide-2026-07-02.html` — English power-user guide for Elias
