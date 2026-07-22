---
name: Survey petition lifecycle — where petitions are created (not here)
description: survey_petition rows are created by ms-valet-service at customer check-OUT time (POST /v2/check-out), not by ms-backoffice-service or the admin CRUD. The check_in column stores the earlier check-in timestamp, not the trigger moment
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
**Non-obvious fact:** `ms-backoffice-service` does **NOT** create rows in `company.survey_petition`. It only reads and updates them (fetch questionnaire, mark `started_at`, insert `survey_answer`). Creation happens in a different microservice.

## Where petitions are created

`ms-valet-service` — `com.corepark.valet.survey.service.impl.SurveyService.handleSurvey(SendSurveyRequestBean)`:

- Called when the valet app registers a customer check-out (from `ValetService.oldCheckout` line 1347 or `ValetService.sendCheckoutNotification` line 1776)
- Looks up the active survey for `(operator, location)` — if none, no petition is created
- If survey exists: generates `surveyPetitionUUID = UUID.randomUUID()`, INSERTs into `company.survey_petition` with `ticket`, `check_in`, `phone_number`, `survey_id`, `created_at`
- Returns a shortlink (`smsService.getSurveyShortLink`) that is sent via SMS to the customer's phone
- Customer opens the SMS link → `frontend-survey` renders using the petitionUuid path segment → answers land back on `ms-backoffice-service` via POST `/survey/questionnaire`

**Repo location:** `/Users/israel/Dev/Back-End/ms-valet-service/src/main/java/com/corepark/valet/survey/service/impl/SurveyService.java`
**Ms-valet DAO with the INSERT:** `com/corepark/valet/survey/dao/impl/SurveyDao.java` — constant `QUERY_SAVE_SURVEY_PETITION`

## Schema constraints on `survey_petition` (verified 2026-07-01)

Beyond obvious NOT NULLs, there is a **composite FK** `fk_survey_petition_parking_service` on `(ticket, check_in, parking_location_id, operator_company_id)` referencing `company.parking_service`. So every petition row must correspond to a real prior check-in row in `parking_service`. This makes it impossible to create "synthetic" preview petitions without either (a) reusing an existing `parking_service` row or (b) creating a fake one first (cascade of more constraints).

Also NOT NULL: `check_in`, `phone_number`.

## Preview / admin visibility path (2026-07-01)

Two attempts made and rolled back before landing on the client-side approach:

1. **First attempt:** `is_preview` boolean column on `survey_petition` + create a preview petition per survey on creation. Rolled back because it required schema change not present in the legacy `.sql` provisioning script.
2. **Second attempt:** New endpoint `POST /valet/survey/preview-petition` on ms-valet-service invoked via Feign from ms-backoffice, creating a synthetic petition per admin click. Rolled back because the composite FK to `parking_service` blocked synthetic petitions.

**Landed approach:** admin visibility renders the survey **client-side in frontend-commerce**, using the `SurveyDetail` returned by `GET /v1/surveys/{surveyUuid}` (new endpoint on ms-backoffice, exposes the pre-existing `SurveyAdminDao.getSurveyDetail` helper). No petition is created; the admin sees the survey inside a dialog labelled "See Survey" — no shareable URL for external preview. If shareable URLs become a real need later, revisit **without touching the petition table** (candidates: reuse an existing `parking_service` without petition, or render `frontend-survey` off a signed static payload).

## How to apply

- If asked "how do surveys reach customers", point to `ms-valet-service.handleSurvey` — not admin CRUD
- If asked "how does the admin preview a survey", it's client-side rendering in commerce (`see-survey-dialog` + `survey-preview` reusable component)
- If tempted to create petitions from ms-backoffice, DON'T — the FK to `parking_service` blocks it
