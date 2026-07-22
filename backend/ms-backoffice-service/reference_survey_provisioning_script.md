---
name: Manual survey provisioning SQL script
description: Location and purpose of the .sql DO block used today to create survey templates per (operator, location)
type: reference
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
**Path:** `/Users/israel/Downloads/PL_SURVEY_QUESTIONS.sql` (Israel's Downloads folder — not versioned in any repo).

Contains a PL/pgSQL `DO $$ ... $$` block that:
- Embeds the whole survey (languages, welcome/completion messages per language, questions with sentences and options) as an inline `JSONB` payload in a variable.
- Iterates a cursor over `(operator_company_id, parking_location_id)` pairs (currently hardcoded to `1,1`, excluding `340`).
- Inserts into all 7 survey tables using **in-memory counters** for per-survey relative IDs. Only `survey.id` uses a real sequence.

**Why:** This is the *only* way to create a survey template today. The admin CRUD being designed will replace it.

**How to apply:** Use as reference for the `POST /surveys` endpoint contract — the JSONB payload shape (`survey.languageIds`, `survey.sentences`, `survey.questions[].options[].sentences[]`) closely maps to what the request body should look like, minus per-survey IDs which the server should generate.

**Known bug in the script:** counters `v_survey_message_id`, `v_supported_survey_language_id`, and `v_question_id` are *not reset* between cursor iterations. It only works reliably for a single (operator, location) at a time. The new backend implementation must reset counters per survey to fix this.
