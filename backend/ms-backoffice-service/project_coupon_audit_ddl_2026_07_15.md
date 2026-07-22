---
name: Coupon feature — audit-history refactor (shipped 2026-07-16)
description: Audit tables (coupon_request, coupon_download) + soft-delete + history endpoint. Shipped end-to-end on 2026-07-16 to feature/staging, both repos. DDL applied in DEV; PROD deploy pending. Supersedes the 07-08 hard-delete-with-USED-lock design
type: project
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
**Status as of 2026-07-17:** shipped end-to-end; PR review iteration re-pushed today.

- **DDL applied in DEV** on 2026-07-16 by the backend team (as `postgres` from AWS Dev Intranet). All four files from `~/Dev/Back-End/ddl/DDL_2026_07_15_COUPONS/` ran clean. `DEV_CLEANUP_LEGACY_COUPON_COLUMNS.sql` executed after the app refactor was verified in DEV — the three legacy columns are gone from DEV as well.
- **Backend** (`ms-backoffice-service`): `feature/coupons` HEAD = `51f83e8` (after 2026-07-17 review-fix commits `5eae7ed` + `51f83e8`). Merged into `feature/staging` as `8808a83`. 58/58 coupon tests green.
- **Frontend** (`frontend-commerce`): commit `be97cf4` on `feature/coupons`, merged into `feature/staging` as `15a14d8`. Build clean.
- **PROD** deploy pending. The DBA gets `DDL_RELEASE_COUPONS_AUDIT_HISTORY.sql` (the consolidated, PROD-grants-only script); FE + BE deploy alongside from `feature/staging` when ready.

## PR review iteration — 2026-07-17 (jvalencia-cp)

Two threads resolved on the PR:

1. **`PmsConnectionServiceImpl.java` — partner CRUD used `ApiException` in a legacy `ServiceException` slice.** Four throws refactored to `ServiceException(MessageCode.X)` in commit `5eae7ed`; four new entries added to `crm.utils.MessageCode`: `BUSINESS_PARTNER_DUPLICATE`, `BUSINESS_PARTNER_INVALID_PHONE`, `BUSINESS_PARTNER_INVALID_COUNTRY`, `BUSINESS_PARTNER_TIMEZONE_LOOKUP_FAILED`. Wire codes preserved (`PARTNER-DUPLICATE`, `PARTNER-INVALID-PHONE`, etc.), FE contract unchanged. See `feedback_prefer_throw_over_return_error_envelope.md` for the "match the file when editing" nuance.
2. **`CLAUDE.md` — DDL section didn't belong there.** 102-line "Schema changes (DDL)" block removed in commit `51f83e8`. Rules archived in `reference_ddl_conventions_pending_relocation.md` pending relocation to Israel's own docs folder.

The `RestExceptionHandlerController` `@ExceptionHandler(ApiException.class)` handler (added in `1d10e26`) was left in place — coupons slice is a genuinely new slice and legitimately uses `ApiException`, so the bridge is still needed.

## The audit-history model (shipped)

Two immutable-row history tables, both with composite FK `(operator_company_id, employee_id)` to `company.employee`:

- **`company.coupon_request`** — one row per provisioning event. `action IN ('CREATE', 'APPEND', 'EDIT', 'DELETE')` (CHECK constraint), signed `vouchers_delta`, `requested_by` INTEGER, `created_at`.
- **`company.coupon_download`** — one row per successful XLSX export. `downloaded_by` INTEGER, `created_at`.

Plus `company.coupon.deleted_at TIMESTAMPTZ` (nullable) for soft-delete. Every operational query filters `WHERE c.deleted_at IS NULL`.

## Backend endpoints (`com.corepark.backoffice.coupon`)

- **`POST /v1/coupons/bulk`** — after each CREATE / APPEND insert, writes `coupon_request` with the corresponding action + signed delta. `CouponEntryRequest.requestedBy` is `Integer` (was VARCHAR).
- **`PUT /v1/coupons/{id}`** — after successful update, writes `coupon_request(action='EDIT', delta=signed)`. `CouponUpdateRequest.requestedBy` is Integer.
- **`DELETE /v1/coupons/{id}?requestedBy=N`** — atomic: `insertCouponRequest(action='DELETE', delta=-(active+used+cancelled))` then `softDeleteCoupon(id)`. **USED-lock rule removed** (`COUPON_HAS_USED_VOUCHERS` / `COUPON-008` deprecated in the enum, kept for backward compat). `requestedBy` comes from a **query param** (not a `Requested-By` header) so the gateway CORS whitelist stays untouched.
- **`GET /v1/coupons/{id}/vouchers/export?downloadedBy=N`** — now `@Transactional(REQUIRED)`. After building the XLSX successfully, writes a `coupon_download` row. Same query-param decision as delete.
- **`GET /v1/coupons`** and **`GET /v1/coupons/{id}`** (implicit via `findSummaryById`) — summary queries **LEFT JOIN LATERAL** both audit tables (was `JOIN LATERAL` — switched 2026-07-22 to decouple audit from operational visibility, see below). Return `CouponSummary` with `requestedBy` + `requestedByName` (creator, from earliest `coupon_request` with `action='CREATE'`) and `lastEventAt` / `lastEventBy` / `lastEventByName` / `lastEventAction` (from a UNION ALL of `coupon_request` + `coupon_download`, newest wins). Dropped `lastDownloaded*` — folded into `lastEvent*`. `requestedBy` and `lastEventBy` are **`Integer` (nullable)** — null for orphan batches.
- **`GET /v1/coupons/{id}/history`** — new endpoint. Returns `List<CouponEventHistory>` = provisioning events UNION downloads, newest first. `id` field intentionally dropped from the DTO (FE tracks by `$index`). Each entry has `action` (`'CREATE' | 'APPEND' | 'EDIT' | 'DELETE' | 'DOWNLOAD'`), `vouchersDelta` (0 for downloads), `requestedBy`, `requestedByName`, `createdAt`.

## Design decisions worth remembering

- **Query params over custom headers** for context-only values on DELETE / GET (e.g. `?requestedBy=`, `?downloadedBy=`). Custom headers require `ms-gateway-service` CORS updates; query params don't. Applies broadly to any small int/string context param on non-body endpoints. Body stays the vehicle for POST/PUT payloads (bulk, update).
- **USED-lock removed on delete**. Soft-delete preserves history regardless of past redemptions; the check was obsolete once `deleted_at` landed. The USED-lock still applies to EDIT for timing/validity/rate-restriction changes.
- **`ACTIVATE` action deferred.** Israel considered a Deactivate/Activate rename (with reactivation flow that clears `deleted_at`) but chose not to touch the DB CHECK constraint again. The CHECK stays at 4 values. UI still says "Delete" — the deactivation semantic is server-side only.
- **Never re-add app-implementation refs to DDL comments.** File 04's Safety section originally listed specific query method names (`insertCoupon`, `QRY_FIND_SUMMARIES_BY_LOCATION`, etc.); rewrite pass on 2026-07-16 dropped those in favor of a generic "the current backend still reads and writes these columns" to match Release 171 style. See DDL style-guide memory in the DDL project.
- **Audit decoupled from operational visibility (2026-07-22).** Original design used `JOIN LATERAL` INNER against `coupon_request/action='CREATE'` in the summary queries. Batches inserted manually into `company.coupon` without a `CREATE` row in `company.coupon_request` were invisible to `GET /v1/coupons` (and also non-editable / non-deletable / no history). Fix shipped in `feature/staging` both repos: the two `JOIN LATERAL` blocks and their `company.employee` joins are `LEFT JOIN` now, and `CouponServiceImpl.updateCoupon` / `deleteCoupon` transparently insert the missing `CREATE` row (attributed to the acting employee from the request payload) before writing the `EDIT` / `DELETE` event. The audit trail is repaired the first time an operator touches the orphan batch, without a separate attribution endpoint. FE renders "Not attributed yet — pick an employee on the next edit" for null-audit rows.

## Java types worth knowing

- **`CouponAction` enum** — `CREATE / APPEND / EDIT / DELETE` — matches the DDL CHECK. `.name()` is what goes into the DB.
- **`CouponEventHistory` DTO** — no `id` field; the history is read-only and immutable, so surrogate ids don't leak to the FE.
- **`CouponSummary`** — the four `lastEvent*` fields span both audit tables. If `lastEventAction === 'DOWNLOAD'`, the latest activity was an export (not a mutation).

## SQL specifics

- **Summary `latest_event` LATERAL** is a `UNION ALL` of `coupon_request(action, requested_by AS actor)` and `coupon_download(action='DOWNLOAD', downloaded_by AS actor)`, ordered by `created_at DESC LIMIT 1`. Then joined to `company.employee` on the composite key.
- **History query** is the same UNION shape (without the LIMIT), ordered by `created_at DESC`. Downloads carry `vouchers_delta = 0`.
- **Every SELECT on `company.coupon`** includes `AND c.deleted_at IS NULL`. Every UPDATE includes it too — soft-deleted rows are unreachable operationally.

## Frontend cross-links

Paired FE work lives in `frontend-commerce` (not `frontend-backoffice` — that migration was deferred). See `project_coupons_module.md` in that project's memory for:

- `ActingEmployeeState` pattern (scoped per operator, localStorage prefill).
- Per-dialog employee picker (form / edit / delete / download / history).
- History dialog timeline UI with per-action colored markers.
- Row simplification to a single "{action} by {name} on {date}" line.

## Cross-links (this project)

- Previous chapter: `project_coupon_progress_snapshot.md` (2026-07-08 hard-delete + USED-lock).
- BE slice detail (pre-refactor): `project_coupon_be_implementation.md`.
- Data model (pre-refactor): `project_coupon_data_model.md`.
- Migration process: `project_migration_process.md`.
- DDL project memory: `~/.claude/projects/-Users-israel-Dev-Back-End-ddl-DDL-2026-07-15-COUPONS/memory/MEMORY.md`.
