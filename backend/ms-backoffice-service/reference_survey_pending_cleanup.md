---
name: Survey — remaining cleanup items after Wave 2 refactor
description: Two items intentionally left out of the 2026-07-07 refactor. Documented so they don't get forgotten and don't get relitigated when the team's next review lands
type: reference
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
After the Wave 2 refactor (bean/ layout + ApiResponse<T> envelope + annotation cleanup, all merged into PR #85 by 2026-07-07), two items were deliberately left out of scope.

## Issue A — OpenAPI annotations missing in SurveyAdminController

**What CLAUDE.md requires:** every REST endpoint documented with springdoc-openapi:

- `@Tag(name, description)` on the controller class
- `@Operation(summary, description[, requestBody])` per endpoint
- one `@ApiResponse(responseCode, description, content)` per documented status
- request/response/error JSON examples externalized to a `<Feature>ApiExamples` class (final class, private constructor via `@NoArgsConstructor(access = AccessLevel.PRIVATE)`, `public static final String` text-block constants per example)

Reference implementation for the pattern:

- `com.corepark.crm.aggregators.controller.RulesController` (cited by CLAUDE.md)
- `com.corepark.backoffice.parkinglayout.controller.ParkingLayoutController` + `ParkingLayoutApiExamples` (cited by CLAUDE.md)
- `com.corepark.backoffice.firebasecleanup.controller.FirebaseCleanupController` + `FirebaseCleanupApiExamples` (added in commit `c2c66a7`, freshest example inside `backoffice/`)

**Current state on survey admin:** `SurveyAdminController` has zero of these annotations. `grep '@Operation\|@Tag\|@ApiResponse\|@ExampleObject' src/main/java/com/corepark/backoffice/survey/controller/*.java` → empty.

**Estimated work:** ~2-3 hours of template-and-fill following `FirebaseCleanupApiExamples`. 7 endpoints × (`@Operation` + 2-4 `@ApiResponse` each) + JSON example constants for the write DTOs.

**Reason deferred:** the migration to `ApiResponse<T>` was already a big cognitive load for one PR (48+ files, 59 tests). OpenAPI docs are additive — they don't change the runtime — so they fit better as a follow-up PR.

## Issue B — `implements Serializable` on beans without functional reason

**Files affected:**
- `bean/dto/SurveyLocationStatus` — `serialVersionUID = 3948572019284756123L`
- `bean/dto/SurveySummary` — `serialVersionUID = 8471029384756192837L`
- `bean/request/MessageRequest` — `serialVersionUID = 2938471056283910472L`
- `bean/request/OptionRequest` — `serialVersionUID = 4726103928475610283L`
- `bean/request/QuestionRequest` — `serialVersionUID = 3819472051937284650L`
- `bean/request/SentenceRequest` — `serialVersionUID = 6829471083926510473L`
- `bean/request/SurveyWriteRequest` — `serialVersionUID = 1029384756192837465L`

**Why they don't need it:** `Serializable` matters for RMI, JMS, HTTP session persistence, or JVM-native cache serialization (Hazelcast, Redis with Java native, etc.). None of that applies to `ms-backoffice-service`:

- Communication is REST with Jackson JSON — never uses Java's `ObjectOutputStream`.
- No session state (stateless).
- No distributed cache with Java serialization.
- Feign clients use HTTP + JSON.

**Estimated work:** ~15 minutes total. Remove `implements Serializable`, remove `serialVersionUID`, remove the `Serializable` import from 7 files.

**Reason deferred:** pre-existing tech debt (not introduced by any recent refactor). Cosmetic only. Low team pressure to fix. Fine to leave for later.

## When these come back

If the team's next review flags either:

- **Issue A**: reach for `FirebaseCleanupApiExamples` as the template — it's the freshest example, same slice pattern, verified working (deployed 2026-07-06).
- **Issue B**: batch delete across all 7 files with a single `find + sed` or just a manual pass.

Neither is a bug — both are convention drift. Deploys are unaffected either way.
