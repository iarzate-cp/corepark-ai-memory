---
name: In service layer, throw domain exceptions instead of returning error envelopes
description: For any new service method, use orElseThrow(new ServiceException/ApiException(MessageCode/ErrorCode.X)) — do not return Response.error() / ApiResponse.error() directly, and do not hand-catch DataAccessException
type: feedback
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
# Prefer throw over return-error-envelope in the service layer

**Rule:** service methods do **not** return error envelopes and do **not** hand-catch `DataAccessException`. Negative business paths throw a domain exception (`ServiceException(MessageCode.X)` in valet-package / legacy slices; `ApiException(ErrorCode.X)` in modern backoffice slices). The registered `@ControllerAdvice` renders the exception into the correct envelope with the correct HTTP status.

**Why:** Israel asked on 2026-07-17 to align the guest vehicle-info service with the exception-based pattern. The original service returned `Response.error(MessageCode.X)` on negative paths and hand-caught `DataAccessException` — it worked, but was inconsistent with the rest of the codebase, forced every controller to derive HTTP status from the response body (`ResponseEntity.status(response.getStatus())`), and duplicated error-envelope construction inside every service method. Throwing is the ecosystem convention, and the exception handlers already emit the correct wire codes.

**How to apply:**

1. Any `Optional<T>` from the DAO turns into `.orElseThrow(() -> new ServiceException(MessageCode.X))` (or `ApiException(ErrorCode.X)` in modern backoffice code).
2. Any explicit "unexpected state" check (e.g., `updateRowCount == 0`, "duplicate found") uses `throw new ServiceException(MessageCode.X)`.
3. **Do not hand-catch `DataAccessException`** — the global handler is already registered to catch it and render `ERR_DB_DATA_ACCESS` uniformly. Adding a local try/catch just duplicates that work and hides the stack trace from central logging.
4. Tests: use `assertThrows(ServiceException.class, ...)` and verify the `MessageCode` via `ex.getMessageCode()` (valet variant stores the enum as a field). For modern slices, assert on `ApiException` + `ex.getErrorCode()`.
5. **Skip only when the service has no business error paths.** If a service is a simple pass-through to the DAO (list endpoint, no `Optional`, no invariants to enforce), there is nothing to throw — leave it alone. Don't invent errors just to fit the pattern. The DB-error path is still covered by the global handler.

**When editing an EXISTING legacy slice, match the FILE, not the CLAUDE.md "modern for new code" rule.**

- CLAUDE.md is explicit: *"Legacy — match it only when editing those slices, don't introduce it in new code"* and *"When a file's surrounding code already follows a local variant, match the file you are editing."*
- Adding a new **method** to an existing `*ServiceImpl.java` counts as *editing that slice*, not *new code*. Even if the method is functionally isolated, throwing `ApiException(ErrorCode.X)` inside a file whose other 20+ throws use `ServiceException(MessageCode.X)` breaks local consistency and reviewers will flag it (confirmed: jvalencia-cp on `feature/coupons` PR, 2026-07-17).
- If the enum lacks the codes you need, **add them to the slice's own `MessageCode` enum** (e.g., add `BUSINESS_PARTNER_DUPLICATE` to `crm.utils.MessageCode`), don't reach for `common.exception.ErrorCode`.
- "New code = new slice" — a genuinely new feature package (no prior file to match). Coupons, vehicle-info-tickets, survey admin are new slices → modern `ApiException`. Partner CRUD added to `PmsConnectionServiceImpl` is an edit → legacy `ServiceException`.

**Precedent from this ecosystem:** on 2026-07-17 the coupon PR had to be refactored (`5eae7ed`) — four `ApiException` throws in `com.corepark.crm.pms.connections.service.impl.PmsConnectionServiceImpl` were converted back to `ServiceException(MessageCode.X)` after the reviewer pointed out the inconsistency with the rest of that file. **Verify by grepping the file first** — if `ServiceException` is the dominant pattern in the file you're about to edit, use it.

**Which exception class to import:**

| Package the service lives in | Exception class | Message enum |
|---|---|---|
| `com.corepark.valet.*` (ms-valet-service) | `com.corepark.valet.utils.ServiceException` | `com.corepark.valet.utils.enums.MessageCode` |
| `com.corepark.partner.*` (ms-valet-service, partner slice) | `com.corepark.partner.exception.ServiceException` | `com.corepark.partner.enums.MessageCode` |
| `com.corepark.backoffice.*` (backoffice slices, legacy) | `com.corepark.backoffice.common.exception.ServiceException` | `com.corepark.backoffice.common.util.MessageCode` |
| `com.corepark.crm.*` (crm slices, legacy) | `com.corepark.crm.exception.ServiceException` | `com.corepark.crm.utils.MessageCode` |
| **new code in `backoffice` or `crm`** (modern convention) | `com.corepark.common.exception.ApiException` | `com.corepark.common.exception.ErrorCode` |

Follow the slice's existing style. **Truly new slices** (no prior file to match) in `backoffice` / `crm` default to the modern `ApiException` + `ErrorCode` per CLAUDE.md.

**Confirmed by Israel — 2026-07-17.** Applied to the vehicle-info feature ecosystem retroactively. The "match the file when editing" nuance was added the same day after the coupons PR review from jvalencia-cp.
