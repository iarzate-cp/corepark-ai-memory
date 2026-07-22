---
name: ApiResponse<T> + ErrorCode + GlobalExceptionHandler — the modern envelope pattern
description: The preferred controller envelope per CLAUDE.md. Verified against com.corepark.common.* on 2026-07-07. Applies to any slice being migrated off the legacy Response + MessageCode pattern
type: reference
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
The team's convention for new endpoints (and for slices being migrated off legacy). Israel is transitioning from FE to BE, so the memory below intentionally leans on FE analogies. Verified against the actual code in `com.corepark.common.*` on 2026-07-07 — not inferred from CLAUDE.md alone.

## Files that make up the pattern

| File | Role |
|---|---|
| `com.corepark.common.response.ApiResponse<T>` | Generic response envelope (data + code + status) |
| `com.corepark.common.exception.ErrorCode` (enum) | Semantic error catalog. Generic entries: `OK`, `NOT_FOUND`, `CONFLICT`, `VALIDATION_FAILED`, `MISSING_HEADER`, `MALFORMED_BODY`, `METHOD_NOT_SUPPORTED`, `INVALID_HEADER`, `BAD_REQUEST`, `DATABASE_ERROR`, `DATABASE_UNREACHABLE`, `INTERNAL_SERVER_ERROR`. **Feature-specific entries are also allowed in the same enum** when the generic semantic code doesn't capture the domain reason — e.g. `FIREBASE_CLEANUP_TICKET_NOT_ORPHAN`, `FIREBASE_CLEANUP_TICKET_NOT_FOUND`, `FIREBASE_CLEANUP_DELETE_FAILED` (added by the firebase-cleanup slice in commit `c2c66a7`). Different from the legacy `MessageCode` in that these are semantic, opt-in per feature, not required. |
| `com.corepark.common.exception.ApiException` | Runtime exception carrying an `ErrorCode` — how services signal domain errors |
| `com.corepark.common.exception.GlobalExceptionHandler` | `@ControllerAdvice` that catches ALL exceptions and renders them as `ApiResponse<Void>` |

## Envelope shape (JSON on the wire)

```json
// success with data:      { "code": "OK", "data": { ... } }
// success no data:        { "code": "CREATED" }         // or "NO-CONTENT"
// validation error:       { "code": "VALIDATION-FAILED", "message": "...", "errors": ["field: reason", ...] }
// domain error:           { "code": "CONFLICT", "message": "Active survey already exists for location 5" }
```

Key design details:
- `int status` in the envelope is `@JsonIgnore`d — it drives HTTP status via `ResponseEntity.status(response.getStatus()).body(response)` but never leaks into JSON. Single source of truth for the HTTP code.
- `data: T` — genericized, so `ApiResponse<SurveyDetail>` is a distinct type from `ApiResponse<List<SurveySummary>>`. Compiler forces the shape.
- `@JsonInclude(NON_NULL)` on the class + `@JsonInclude(NON_EMPTY)` on `errors` — null/empty fields drop out of the JSON.

## Factory methods (never call `new ApiResponse(...)` directly)

```java
ApiResponse.success(data)                    // 200 + data
ApiResponse.success(data, "message")         // 200 + data + human message
ApiResponse.created()                        // 201, no body
ApiResponse.noContent()                      // 204, no body
ApiResponse.error(ErrorCode.NOT_FOUND)       // status/code/message from the enum
ApiResponse.error(ErrorCode.VALIDATION_FAILED, listOfErrors)
ApiResponse.error(status, code, message)     // raw form, used by GlobalExceptionHandler
```

## Service layer signals errors by throwing

```java
public SurveyDetail getSurveyDetail(int op, int loc, UUID uuid) {
    SurveyDetail detail = dao.findByUuid(op, loc, uuid);
    if (detail == null) {
        throw new ApiException(ErrorCode.NOT_FOUND, "Survey " + uuid + " not found");
    }
    return detail;
}
```

Never return `null` or `Optional.empty()` on domain errors — throw. Controllers stay clean; the global handler translates.

**Detail message carries specificity.** In legacy you had to invent `ERR_SURVEY_005` to differentiate one conflict from another; here `CONFLICT` stays the semantic code and the `message` string carries the specific reason. Create a *new* `ErrorCode` only when the FE needs to branch on the error type; otherwise reuse a semantic code + detail message.

## Controller shape

```java
@RestController
@RequestMapping("/v1/surveys")
@RequiredArgsConstructor
public class SurveyAdminController {

    private final SurveyAdminService service;

    @GetMapping("/{surveyUuid}")
    public ResponseEntity<ApiResponse<SurveyDetail>> getSurvey(
            @RequestHeader("Operator-Id") int operatorId,
            @RequestHeader("Location-Id") int locationId,
            @PathVariable UUID surveyUuid) {

        SurveyDetail detail = service.getSurveyDetail(operatorId, locationId, surveyUuid);
        ApiResponse<SurveyDetail> response = ApiResponse.success(detail);
        return ResponseEntity.status(response.getStatus()).body(response);
    }
}
```

Do NOT in the controller:
- try/catch service exceptions → the `GlobalExceptionHandler` handles them
- validate headers manually → Spring throws `MissingRequestHeaderException`, handler translates to `MISSING-HEADER`
- validate JSON parse → Spring throws `HttpMessageNotReadableException`, handler translates to `MALFORMED-BODY`
- parse `MessageCode` to `HttpStatus` (that's the legacy anti-pattern)

## GlobalExceptionHandler catches (as of 2026-07-07)

Registered at `LOWEST_PRECEDENCE` so any more-specific `@ControllerAdvice` wins:

| Exception | → ErrorCode / HTTP |
|---|---|
| `ApiException` | uses its own `ErrorCode` (any status the service picked) |
| `MethodArgumentNotValidException` (`@Valid` failed) | `VALIDATION_FAILED` (400) + field errors |
| `MissingRequestHeaderException` | `MISSING_HEADER` (400) |
| `HttpMessageNotReadableException` | `MALFORMED_BODY` (400) |
| `HttpRequestMethodNotSupportedException` | `METHOD_NOT_SUPPORTED` (405) |
| `NoResourceFoundException` | `NOT_FOUND` (404) |
| `HandlerMethodValidationException` | `INVALID_HEADER` (400) |
| catch-all `Exception` | `INTERNAL_SERVER_ERROR` (500) |

No feature-specific `DuplicateKeyException` / `DataIntegrityViolationException` handlers here — those are legacy leftovers in `RestExceptionHandlerController`. If you need to translate a DB exception to a domain error in the new pattern, catch it in the service and re-throw as `ApiException`.

## Contrast with the legacy pattern (still in use in `SurveyController`, `SurveyAdminController`, most of backoffice)

| Aspect | Legacy (`Response` + `MessageCode`) | Modern (`ApiResponse<T>` + `ErrorCode`) |
|---|---|---|
| Payload type | `Response` (blob object) | `ApiResponse<T>` (typed generic) |
| HTTP status | `HttpStatus.valueOf(MessageCode.fromCode(response.getCode()))` (string reparse) | `response.getStatus()` (int) |
| Error codes | `MessageCode.ERR_SURVEY_005` (feature-scoped, dozens per feature) | `ErrorCode.CONFLICT` (semantic, ~12 total) |
| Service error signal | `throw new ServiceException(MessageCode.ERR_SURVEY_005)` | `throw new ApiException(ErrorCode.CONFLICT, "detail")` |
| Handler | `RestExceptionHandlerController` with 10+ ad-hoc handlers | `GlobalExceptionHandler` — 7 semantic handlers, reusable |
| FE contract | `{ code: "ERR_SURVEY_005", errors: [...] }` | `{ code: "CONFLICT", data/message: ... }` |

**When to migrate a slice:** whole-slice migration is preferred (never mix envelopes inside one feature). Include the FE change in the same PR sequence — the JSON contract changes shape.

## Reference controllers in the repo

- **Modern (follow these):**
  - `com.corepark.crm.aggregators.controller.RulesController` (cited by CLAUDE.md)
  - `com.corepark.backoffice.firebasecleanup.controller.FirebaseCleanupController` — full slice migration from legacy to modern; commit `c2c66a7` (2026-07-06) is the canonical diff to copy when migrating another slice
- **Legacy (do NOT copy for new work):** `com.corepark.backoffice.survey.controller.SurveyAdminController`, `SurveyController` (both scheduled for migration when the FE contract change can be coordinated)
