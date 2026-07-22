---
name: Survey data model — tables, hierarchy, non-obvious constraints
description: Tree hierarchy under company.survey, per-survey relative IDs, multilingual pattern, preview petitions, and inferences about versioning/soft-delete
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
**Table hierarchy** (schema `company`):

```
survey (id [seq], uuid, operator_company_id, parking_location_id, survey_type_id, created_at, deactivated_at)
├── supported_survey_language (id, survey_id, language_id)
├── survey_message (id, operator, location, survey_id, language_id,
│                   welcome_message, welcome_submessage,
│                   completition_message, completition_submessage)
└── survey_question (id, operator, location, survey_id, question_type_id, optional)
    ├── survey_question_sentence (id, operator, location, survey_id, question_id, language_id, sentence)
    └── survey_question_option (id, operator, location, survey_id, question_id, open_answer)
        └── survey_question_option_sentence (id, operator, location, survey_id, question_id, option_id, language_id, sentence)
```

Answer/petition tables (populated by end-user flow only):
- `survey_petition (id, uuid, operator_company_id, parking_location_id, ticket, check_in, phone_number, survey_id, created_at, started_at)` — one row per customer invitation. No `is_preview` column — a preview mechanism was designed and then rolled back on 2026-07-01.
- `survey_answer (id, operator_company_id, parking_location_id, petition_id, survey_id, question_id, option_id, answer_text)`

**Non-obvious facts:**

1. **IDs are relative per survey.** `question_id`, `option_id`, `message_id`, `supported_survey_language.id`, and sentence IDs start at 1 within each `survey_id`. Only `survey.id` and `survey_petition.id` use global sequences (`company.survey_id_seq`, `company.survey_petition_id_seq`). The admin CRUD's `SurveyAdminDaoImpl` uses `i + 1` ordinals from `List.size()` in Java — no in-memory counters like the old `.sql` script.
2. **`company.survey` has its own `uuid`** distinct from `survey_petition.uuid`. Terminology: `surveyUuid` = template, `petitionUuid` = individual invitation (may be preview).
3. **Soft-delete/versioning via `company.survey.deactivated_at`.** One active survey per (operator, location) at a time — enforced in `SurveyAdminServiceImpl.rejectIfActiveSurveyExists`. Editing = create new + deactivate old (versionado automático — not yet implemented).
4. **`operator_company_id` + `parking_location_id` are denormalized** onto every descendant table (question, option, sentence rows). Probably for query performance and scoping.
5. **`survey_answer.answer_text` exists but is always NULL today.** Reserved for `openAnswer=true` options that would allow free text; not implemented in the current submit path.
6. **DTO field additions (2026-07-01):**
   - `Question.optional` — the schema had `sq.optional` all along, but the consumer `SurveyQuestionnaire` DTO never exposed it. Now populated in the admin `getSurveyDetail` extractor; consumer flow leaves it as default false.
   - `Option.openAnswer` — same story with `sqo.open_answer`.

**Catalog values observed:**
- `surveyTypeId = 1` (customer satisfaction — only value observed)
- `questionTypeId`: `1 = Rate` (star rating), `2 = RadioButton`
- `languageId`: `1 = English`, `2 = Spanish`, `5 = French` (from `custom.cat_language`)

**How to apply:** when designing admin CRUD contracts, treat the survey tree as immutable-per-version once answers exist. Reordering/deleting a question that already has answers would break referential integrity if IDs are reused. Prefer append-only within a version, or force a new version (which is what the versionado plan already does).
