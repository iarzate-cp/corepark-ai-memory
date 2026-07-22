---
name: Survey — Lombok annotations saga and canonical answer (updated 2026-07-08)
description: Analysis of @EqualsAndHashCode / @ToString on survey DTOs. 6 @EqualsAndHashCode stay (RowMapper dedup, functionally required); all @ToString removed and the two verbose logs that used them tightened. Reference for defending these choices in future reviews
type: project
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
Feedback landed on 2026-07-07 from a team member reviewing PR #85: *"vi varias anotaciones como estas, pregúntale a claudio si realmente se usan — `@EqualsAndHashCode(of = "uuid")`, `@ToString`, `@EqualsAndHashCode(of = {"id", "questionId"})`, `@ToString(of = {"id", "questionId", "sentences"})`"*. This memory documents the analysis so we don't relitigate.

## History

- **Commit `53ae7c0`** (original consumer flow): DTOs `Question`, `Option`, `Sentence`, etc. were created with `equals()` / `hashCode()` **hand-written** — ~15-20 lines each. Not decorative — the `SurveyDaoImpl` RowMapper uses them as `Set` elements and `Map` keys.
- **Commit `13c25d4`** (admin CRUD original): same pattern, hand-written equals/hashCode.
- **Commit `2887bfc` (2026-07-06)**: Israel refactored to Lombok — replaced hand-written equals/hashCode with `@EqualsAndHashCode(of = "id")` annotations. **Semantically identical** to the hand-written versions; Lombok generates the same bytecode.
- **Commits `687e0ad` + `f381453` (2026-07-07)**: my Phase 1 and Phase 2 refactors did NOT touch these DTOs' contents (only moved packages and rewrote the controller/service/tests).

The team's review confusion is legitimate: annotations declare, but usage lives in another file. Without grepping the callers, `@EqualsAndHashCode(of = "id")` looks decorative on a data class.

## Answer: 6 stay, 3 removed

### STAY — used by the RowMapper's dedup algorithm

Cartesian-join queries return duplicate rows (one per unique combination of question/option/sentence). The RowMapper folds them into a proper tree using `Set` and `Map`. Without `equals`/`hashCode`, `Set.contains(aux)` compares by identity, and every iteration produces a "new" object → dedup breaks → tree ends up with duplicates.

| DTO | Annotation | Where used |
|---|---|---|
| `Question` | `@EqualsAndHashCode(of = "id")` | `SurveyDaoImpl:108,111` (`Set<Question>` + `Map<Question, Question>` key), `SurveyAdminDaoImpl:484,486` |
| `Option` | `@EqualsAndHashCode(of = {"id", "questionId"})` | `SurveyDaoImpl:112,143` (`Set<Option>` + `Map<Option, Option>` key). Composite key because `id` is relative-per-question in the data model — same `id` can repeat across questions |
| `Sentence` | `@EqualsAndHashCode(of = "id")` | `SurveyDaoImpl:157,160` (`Set<Sentence>` + `.contains()`) |
| `SurveyMessage` | `@EqualsAndHashCode(of = "id")` | `SurveyDaoImpl:107,138`, `SurveyAdminDaoImpl:483` |
| `SupportedLanguage` | `@EqualsAndHashCode(of = "id")` | `SurveyDaoImpl:106`, `SurveyAdminDaoImpl:482` |
| `QuestionAnswer` | `@EqualsAndHashCode(of = "questionId")` | `SurveyPetitionSubmit.answers` and `PetitionQuestionnaireSubmitRequest.answers` are `Set<QuestionAnswer>` — dedup by question |

### REMOVED — never entered a Set/Map, never compared with .equals()

| DTO | Annotation | Removed in |
|---|---|---|
| `SurveyDetail` | `@EqualsAndHashCode(callSuper = false)` | `4fc5124` |
| `SurveyQuestionnaire` | `@EqualsAndHashCode(of = "uuid")` | `25d9d66` |
| `SurveyQuestionnaireResponse` | `@EqualsAndHashCode(callSuper = false)` | `90d2345` |

The `callSuper = false` on the response classes was a Lombok convention when the class `extends Response` (legacy envelope). Under `ApiResponse<T>`, `SurveyDetail` no longer extends anything, so `callSuper = false` had no target.

## @ToString — all removed (2026-07-08)

**Original position (superseded)**: kept everywhere as zero-cost debug convenience.

**Correction on 2026-07-08**: the review comment on `Option.@ToString` triggered a deeper grep. My first pass missed that two `LOGGER.info` calls in `SurveyServiceImpl` were dumping the full `SurveyQuestionnaire` and `PetitionQuestionnaireSubmitRequest` trees at INFO level:

- `SurveyServiceImpl:45` — `LOGGER.info("Survey questionnaire pulled {}", surveyQuestionnaire)` — transitively called `.toString()` on `SurveyQuestionnaire` → `SupportedLanguage`, `SurveyMessage` → `Message`, `Question` → `Sentence`, `Option` → `Sentence`. 3–5 KB of nested output per request for a modest survey.
- `SurveyServiceImpl:82` — `LOGGER.info("Submitting survey questionnaire {}", request)` — same shape for `PetitionQuestionnaireSubmitRequest` → `QuestionAnswer`.

The right fix wasn't to keep `@ToString` "just in case"; it was to fix the noisy logs (log the UUID, not the payload) and delete the annotations. Applied in two staged commits:

- `ad1ec37 chore(survey): drop unused @ToString from three orphan DTOs` — removed from `SurveyPetitionSubmit`, `SurveyPetition`, `SurveyQuestionnaireResponse` (never entered a log path directly or transitively).
- `3698d36 chore(survey): tighten questionnaire logs and drop now-unused @ToString` — rewrote both log statements to emit only the UUID, then removed `@ToString` from `Message`, `Option`, `Question`, `QuestionAnswer`, `Sentence`, `SupportedLanguage`, `SurveyMessage`, `SurveyQuestionnaire`, `PetitionQuestionnaireSubmitRequest`.

**Zero `@ToString` remaining in the survey slice as of 2026-07-08.** Tests: 59/59 green after both commits.

**Lesson**: SLF4J `{}` interpolation invokes `.toString()` invisibly. `grep`-ing for `.toString()` calls or literal type names in log statements misses it — variable names hide the type. To audit correctly, follow the variable back to its declaration and trace the transitive graph of field types. See the "Rule to apply on future reviews" section for the corrected grep pattern.

## The mental model to reuse for future annotations reviews

Two distinct roles get conflated in review comments:

1. **Response envelope (external)** — `extends Response`, `@EqualsAndHashCode(callSuper = false)`, `serialVersionUID`. These are cosmetic to the legacy envelope pattern. Under `ApiResponse<T>` they're residue and should be removed when found.
2. **Internal domain objects (motor)** — `@EqualsAndHashCode(of = "id")` on nested DTOs that live inside `Set`/`Map` in DAO code. These are **functional requirements** of the dedup algorithm. Independent of the response envelope; do not remove.

Analogy Israel found useful: *"The plate is the envelope; the chef in the kitchen distinguishing salt from sugar is the internal motor. Changing the plate doesn't affect that the chef still needs to tell ingredients apart."*

## Rule to apply on future reviews

Before recommending removal or preservation of a Lombok annotation on a DTO:

**For `@EqualsAndHashCode`**: grep for the class name across `**/*.java` and check whether it appears as a generic type in `Set<X>`, `Map<X, ?>`, `Map<?, X>`, or in `.contains(x)`, `.equals(y)` calls. If yes → equals/hashCode is required. If no → the annotation is dead weight.

**For `@ToString`** (updated 2026-07-08): the earlier version of this rule (grep the type name in log statements) is insufficient because SLF4J `{}` interpolation hides the argument type behind a variable name. Correct procedure:

1. Grep every `log.` / `LOGGER.` call in the slice.
2. For each call, note the variable(s) passed after the format string.
3. Follow each variable to its declaration to determine its type.
4. If any variable is one of the DTOs under review, `@ToString` on it is a direct caller.
5. If a DTO field of another logged DTO is a collection or non-primitive → `@ToString` is a transitive caller (via the parent's toString cascade).

If NEITHER direct nor transitive callers exist, `@ToString` is dead weight. Consider ALSO whether the log line itself is worth keeping — dumping large object trees at INFO level is usually a smell that should be rewritten to emit the identifier only.

Corollary: newly-created DTOs that don't hit dedup collections don't need `@EqualsAndHashCode` at all. Don't sprinkle it "just in case". Same for `@ToString` — only add it when a log statement actually needs it, and prefer logging identifiers over payloads.
