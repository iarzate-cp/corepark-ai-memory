---
name: Firebase orphan ticket cleanup feature — shipped to dev 2026-07-06
description: Cross-repo maintenance utility to detect and delete tickets left in the Firebase carlist after DB checkout. Feature shipped to feature/staging (dev) on 2026-07-06 across four repos. Two blockers resolved; a set of follow-ups deferred for a later iteration
type: project
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---

# Firebase orphan-ticket cleanup

## Status

**Shipped to `feature/staging` (dev) on 2026-07-06** across three repos plus the frontend. Both original blockers resolved during this iteration. A well-defined list of follow-ups is deferred — see below.

## Business context

Tickets exist in two systems: Postgres (`company.parking_service`, source of truth) and Firebase Realtime DB (a "carlist" of currently active tickets). Roughly every 2 months a ticket that was checked out in Postgres gets left behind in the Firebase carlist — an "orphan". Root cause is that `ms-valet-service` publishes the CHECK-OUT event via RabbitMQ with an HTTP fallback to `ms-firebase-service`; if both paths silently fail after the DB transaction has already committed, Firebase never gets the delete and the ticket lingers. This feature is a manual cleanup tool for backoffice ops, not a fix for the upstream leak.

**Why:** Ops need to purge these stale entries because they mislead the valet app into showing "active" cars that already left. The upstream fix in ms-valet-service is deliberately out of scope for this feature.

**How to apply:** Frame every design decision as "maintenance utility for occasional manual cleanup" — not a background job, not a scheduled sweep, not a real-time reconciliation. That framing is what shapes the current UX (dialog opened from a per-location row in the UUID table, one ticket at a time, double confirmation before delete).

## Repos and their roles

- **`ms-backoffice-service`** (`~/Dev/Back-End/ms-backoffice-service`) — owns the `firebasecleanup` slice. Exposes `GET /backoffice/firebase-cleanup/tickets/check` and `DELETE /backoffice/firebase-cleanup/tickets` at the gateway (which strips `/backoffice` — the controller's `@RequestMapping` is `/firebase-cleanup`). Uses `RestTemplate` to call ms-firebase-service.
- **`ms-firebase-service`** (`~/Dev/Back-End/ms-firebase-service`) — the ONLY service allowed to speak the Firebase Admin SDK. Exposes `GET /ticket/carlist` and `DELETE /ticket` for the internal consumer.
- **`ms-valet-service`** (`~/Dev/Back-End/ms-valet-service`) — writes to the carlist on check-in and removes on check-out via RabbitMQ. Root cause of orphans; not modified by this feature.
- **`frontend-commerce`** (`~/Dev/frontend-commerce`) — `OrphanTicketDialog` in `shared/components/orphan-ticket-dialog/`, opened from the UUID table's per-row action menu.

## The Firebase carlist — semantic that took investigation to establish

Path in Firebase Realtime DB: `/parkingServices/operatorCompany/{opId}/parkingLot/{locId}/ticket/`.

Tickets are **removed from this node** the moment they transition to status `COMPLETE` (check-out). So an active ticket in the carlist inherently has no meaningful `checkOut` field — the deletion is what marks the checkout, not a flag being set. Consequence: the semantic "exists in Firebase AND checkOut is not complete" is equivalent to "exists in the carlist listing". Membership is the correct signal — do not add code that inspects a `checkOut` field on a carlist entry.

## Response shape (post-refactor)

Backoffice returns the modern `ApiResponse<T>` envelope:

```json
{ "code": "OK", "data": { "operatorCompanyId": …, "ticket": "…", "checkedOutInDb": true, "existsInFirebase": true, "orphan": true } }
```

Frontend unwraps `.data` in `FirebaseCleanupService` via `.pipe(map(r => r.data))`, so consumers of the service see a flat `TicketCheckData`. This aligns with the frontend CLAUDE.md convention.

## Blockers that were resolved this iteration

1. **`service.firebase.url` config in dev/prod** — resolved by commit `9609c1c chore: define service.firebase.url for dev and prod`. Dev uses `http://ms-firebase-service.${NAMESPACE}`, prod uses `http://ms-firebase-service.internal-corepark.com` (matches `ms-valet-service`'s Feign convention).
2. **`ERR_FIREBASE_CLEANUP_003` never triggered** — resolved as part of the big refactor `c2c66a7 refactor(firebase-cleanup): align slice with backoffice CLAUDE.md conventions`. `MsFirebaseClient.deleteTicketFromCarlist` now returns a `DeleteTicketResult { OK, NOT_FOUND, FAILED }` enum by inspecting `HttpClientErrorException` body for `code: "TICKET-04"`; the service maps NOT_FOUND to `ApiException(ErrorCode.FIREBASE_CLEANUP_TICKET_NOT_FOUND)` (404).

## Additional fixes made this iteration

- **Controller path aligned with gateway StripPrefix=1** (`e8da4d8`). Was `/backoffice/firebase-cleanup`, now `/firebase-cleanup`. The `/backoffice` segment is stripped by the gateway before reaching the backoffice.
- **Full CLAUDE.md convention refactor** (`c2c66a7`): Lombok DI + `@Slf4j`, migration from legacy `Response`/`MessageCode` to `ApiResponse<T>` + `ErrorCode` in `com.corepark.common.*`, OpenAPI documentation with externalized `FirebaseCleanupApiExamples`, semantic error code names.
- **Local iteration section added to backoffice CLAUDE.md** (`14c7723`) — captures the atomic-task workflow rules.
- **`DELETE /ticket` endpoint added to ms-firebase-service** (`a4b790b`) — the endpoint the backoffice depends on. Was uncommitted local work when the feature started; formalized during this iteration.

## Deferred follow-ups (open items for the next iteration)

- **Tests**: no unit tests exist for the backoffice `firebasecleanup` package. Frontend spec is `pending()`. Backend needs JUnit 4 + Mockito `FirebaseCleanupControllerTest`, `FirebaseCleanupServiceImplTest`, `FirebaseCleanupDaoImplTest`, `MsFirebaseClientTest`.
- **Frontend `NotificationService` migration**: `OrphanTicketDialog` still uses `MatSnackBar` directly. Should use `notifyHttpError` per the `feedback_notification_service` memory.
- **Frontend refresh post-delete**: the dialog closes immediately after a successful delete. Consider resetting the form state and letting the user run another check without reopening.
- **Frontend i18n**: dialog strings ("Find orphan ticket — {location}", "Checked out in DB", the confirmation checkbox copy, etc.) are all hardcoded in English. Should live in `assets/i18n/en/*.json` with keys in a `core/i18n/*-i18n.ts` constant map.
- **Backend timeout differentiation**: `ms-firebase-service` returns `500 SEARCH-TIME-OUT` on 30-second carlist reads. The backoffice currently collapses that into `ERR_FIREBASE_CLEANUP_002`. Worth its own error code so the FE can show a specific message ("Firebase is slow, retry") vs a generic failure.

## Deploy sequence used (works for next iteration)

Firebase → Backoffice → Frontend, because:
- Backoffice's delete flow calls Firebase's `DELETE /ticket`. Deploying backoffice first means any delete attempt on dev returns 404 until firebase catches up.
- Frontend's new `.data` extraction requires backoffice to be on the new `ApiResponse<T>` shape. Deploying frontend first means the dialog renders undefined values until backoffice catches up.

For each repo: `git checkout feature/staging` → `git pull --no-rebase origin feature/staging` → `git merge feature/backoffice-firebase-orphan-ticket-check` → resolve conflicts → `git push`.

## Cross-cutting reference memories that supported this feature

- `reference_backend_microservices.md` — the gateway StripPrefix pattern, ports, service discovery convention.
- `reference_firebase_infrastructure.md` — Firebase dev/prod URLs, carlist path.
- `reference_backoffice_backend_claudemd.md` — pointer to the backoffice CLAUDE.md and its conventions.
- `feedback_atomic_task_workflow.md` — the rule that emerged from this feature's iteration.
- `feedback_verify_git_state.md` — do not trust working-tree state when reporting cross-repo API existence.
