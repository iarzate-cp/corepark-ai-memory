---
name: Survey endpoints — full catalog (consumer legacy + admin modern envelope)
description: /survey/questionnaire (consumer, still Response + MessageCode) + /v1/surveys/* (admin, migrated to ApiResponse<T> + ErrorCode on 2026-07-07). Error code catalog reflects both envelopes
type: project
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
Two personas, two namespaces, **two envelopes** as of 2026-07-07.

## Consumer flow — `/survey/questionnaire`

Owned by `SurveyController` + `SurveyService` + `SurveyDao`. Consumed by `frontend-survey`. **Do not overload with admin concerns.**

- **`GET  /survey/questionnaire?uuid={petitionUuid}`** — fetch questionnaire tree by petition UUID. No headers beyond `Content-Type`.
- **`POST /survey/questionnaire`** — submit answers. Body `{ operatorCompanyUUID, parkingLocationUUID, surveyPetitionUUID, answers[] }`.

Reads use joins that ignore `deactivated_at` (petition-based) — customers with in-flight petitions can still submit even if the survey has been archived. Preserves customer experience during admin edits.

## Admin flow — `/v1/surveys`

Owned by `SurveyAdminController` + `SurveyAdminService` + `SurveyAdminDao`. Consumed by `frontend-commerce` `/settings/survey`.

**Full CRUD, tenant-isolated** (branch `feature/survey-admin-crud`):

| Verb | Path | Headers | Body | Purpose |
|---|---|---|---|---|
| GET | `/v1/surveys/locations-status` | `Operator-Id` | — | Operator's locations + active survey summary (or null per location) |
| GET | `/v1/surveys/history` | `Operator-Id` + `Location-Id` | — | Full history (active + archived), ordered by `created_at DESC` |
| GET | `/v1/surveys/{surveyUuid}` | `Operator-Id` + `Location-Id` | — | Full survey tree. Filtered by tenant in SQL — wrong-tenant returns `ERR_SURVEY_001` |
| POST | `/v1/surveys` | `Operator-Id` + `Location-Id` | `SurveyWriteRequest` | Create — inserts 7 tables in one `@Transactional`. Guarded by app-level check + DB unique index |
| PATCH | `/v1/surveys/{surveyUuid}` | `Operator-Id` + `Location-Id` | `SurveyWriteRequest` | Edit via versioning — deactivate old + create new atomically. Tenant-filtered |
| DELETE | `/v1/surveys/{surveyUuid}` | `Operator-Id` + `Location-Id` | — | Soft-delete via `deactivated_at = NOW()`. Idempotent. Tenant-filtered |
| POST | `/v1/surveys/{surveyUuid}/reactivate` | `Operator-Id` + `Location-Id` | — | Atomic swap: deactivate current active (if any) + reactivate target. Tenant-filtered |

**Semantics of Update:** versioning over in-place edit. Every PATCH creates a new row in `company.survey` and archives the previous one. Preserves historical `survey_answer` data associated with the old survey.

**Semantics of Reactivate:** atomic swap. If the location has an active survey, it gets deactivated in the same transaction as the target is reactivated. Frontend surfaces this via a warning notice (`hasActiveSurvey: true` in dialog data). The "one active per (op, loc)" invariant is preserved.

**Cross-tenant semantics:** any request with a `surveyUuid` belonging to a different `(op, loc)` than the request headers returns `ERR_SURVEY_001` (404-like). Deliberate — masks resource existence, best security posture.

## DB constraint

**`uq_survey_active_survey`** — unique partial index on `(operator_company_id, parking_location_id) WHERE deactivated_at IS NULL`. Exists in dev/staging/prod (DBA-confirmed 2026-07-02). Closes TOCTOU races on create/reactivate at the DB layer. Application check (`existsActiveSurveyForLocation`) is redundant with it but kept for fast 409 without hitting the DB in the 99% case.

## Error code catalog — **as of 2026-07-07**

**Admin flow uses `com.corepark.common.exception.ErrorCode`** (modern envelope). **Consumer flow still uses `com.corepark.backoffice.common.util.MessageCode`** (legacy).

Admin (`/v1/surveys/*`):

| Code | HTTP | Trigger | Source |
|---|---|---|---|
| `SURVEY_NOT_FOUND` | **404** | Survey not found / cross-tenant | DAO `getSurveyDetail` → `EmptyResultDataAccessException`; service on `deactivateSurvey/reactivateSurvey` rowCount == 0 |
| `SURVEY_ACTIVE_ALREADY_EXISTS` | 409 | Active survey already exists for location | service `rejectIfActiveSurveyExists`; also service catch on `DuplicateKeyException` from `uq_survey_active_survey` |
| `SURVEY_LANGUAGES_REQUIRED` | 400 | `languageIds` empty/null | service `validateBusinessRules` |
| `SURVEY_QUESTIONS_REQUIRED` | 400 | `questions` empty/null | service `validateBusinessRules` |
| `SURVEY_MESSAGES_LANGUAGE_COVERAGE` | 400 | Messages don't cover all languages | service |
| `SURVEY_QUESTION_LANGUAGE_COVERAGE` | 400 | Question sentences don't cover all languages | service |
| `SURVEY_OPTION_LANGUAGE_COVERAGE` | 400 | Option sentences don't cover all languages | service |
| `SURVEY_QUESTION_OPTIONS_REQUIRED` | 400 | Question has zero options | service |
| `SURVEY_INVALID_REFERENCE` | 400 | FK/integrity violation | service catch on `DataIntegrityViolationException` |

Consumer (`/survey/questionnaire`):

| Code | HTTP | Trigger | Source |
|---|---|---|---|
| `ERR_SURVEY_001` | 400 | Survey not found | consumer DAO |
| `ERR_SURVEY_002` | 400 | Petition UUID missing | consumer service |
| `ERR_SURVEY_003` | 400 | Answer count mismatch | consumer service |
| `ERR_SURVEY_004` | 400 | Petition already answered | consumer service |

## Handler chain — as of 2026-07-07

**Admin flow** uses `com.corepark.common.exception.GlobalExceptionHandler`. There are no survey-specific handlers there — the service catches `DuplicateKeyException` / `DataIntegrityViolationException` locally and re-throws as `ApiException(ErrorCode.SURVEY_*)`, which `GlobalExceptionHandler.handleApiException` renders.

**Consumer flow** still uses the legacy `com.corepark.backoffice.common.controller.RestExceptionHandlerController` with a `ServiceException` handler that maps `MessageCode` → `Response`.

Spring picks the most specific `@ExceptionHandler` per exception type. `ApiException.class` beats `Exception.class` (the legacy catch-all), so admin flow errors don't get intercepted by the legacy handler even though both `@ControllerAdvice`s are active in the same context.

## Rolled back on 2026-07-01

- `POST /v1/surveys/preview-petition` (Feign to `ms-valet-service`) — replaced by client-side render because `survey_petition` has composite FK to `parking_service` on `(ticket, check_in, parking_location_id, operator_company_id)` that blocks synthetic petitions. See `project_survey_petition_lifecycle.md`.
- `ERR_SURVEY_012` — was previously reserved for "preview petitions cannot accept answers"; that mechanism was removed with the rollback. The code was later reused (2026-07-02) for the generic FK/integrity violation described above.

## Reference DAO helpers (for future admin endpoints)

Reuse from `SurveyAdminDao`: `existsActiveSurveyForLocation(op, loc)`, `getSurveyDetail(op, loc, uuid)`, `deactivateSurvey(op, loc, uuid)`, `deactivateActiveForLocation(op, loc)`, `reactivateSurvey(op, loc, uuid)`, `getLocationHistory(op, loc)`.

All admin mutations MUST be `@Transactional(REQUIRED)` on the service layer. Validation goes in the service (`validateBusinessRules`), not the controller (controller only does BindingResult for JSR-380).
