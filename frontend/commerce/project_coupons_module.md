---
name: Coupons module — audit-history refactor shipped 2026-07-16
description: Coupons live in frontend-commerce at /settings/coupons with a full audit trail across coupon_request + coupon_download, per-dialog employee attribution, row-level History timeline, and soft-delete. Shipped in place; the planned migration to frontend-backoffice was postponed indefinitely.
type: project
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---

## Shipped state (2026-07-16)

Commits: `be97cf4` on `feature/coupons`, merged into `feature/staging` as `15a14d8` (both pushed to origin). Paired backend commit `2e24a13` on `ms-backoffice-service` `feature/coupons`, merged as `1587c86`.

The audit-history refactor landed **in place** in `frontend-commerce`. The move to `frontend-backoffice` did NOT happen and is postponed indefinitely.

## Architecture

### New definitions and services

- `src/app/core/definitions/employee.d.ts` — `Employee { employeeId: number; fullName: string }` + `EmployeeCatalogueResponse`.
- `src/app/core/services/employees-service.ts` — `list(operatorCompanyId)` calls `POST /backoffice/catalogue/employee` with body `{ operatorCompanyId }`. Backend returns `{ code, message, employees[] }` — legacy shape, NOT the standard `ApiResponse<T>` envelope.
- `src/app/core/states/acting-employee-state.ts` — Signal-based state. `ensureLoadedForOperator(id)` caches the employee list per operator. The last-selected `employeeId` persists in `localStorage` under key `coupons.actingEmployeeId` for cross-dialog prefill. `reset()` clears everything on operator change.

### Coupon definitions (`src/app/core/definitions/coupon.d.ts`)

- `CouponAction = 'CREATE' | 'APPEND' | 'EDIT' | 'DELETE' | 'DOWNLOAD'`.
- `CouponEntry.requestedBy: number` (was `string`).
- `CouponSummary` fields: `requestedBy: number`, `requestedByName: string`, `lastEventAt: string`, `lastEventBy: number`, `lastEventByName: string`, `lastEventAction: CouponAction`. Dropped `downloadedAt` — replaced by `lastEvent*` which spans coupon_download too.
- `CouponUpdatePayload.requestedBy: number`.
- New `CouponEventHistory` (no id — the timeline dialog uses `track $index`).

### Coupons service (`src/app/core/services/coupons-service.ts`)

- `deleteCoupon(op, loc, id, requestedBy)` sends `?requestedBy=` as a **query param**.
- `exportVouchers(op, loc, id, downloadedBy)` sends `?downloadedBy=` as a **query param**.
- New `getHistory(op, loc, couponId)` calls `GET /backoffice/v1/coupons/{id}/history`.

### Path setter (`src/app/core/utils/path-service-setter.ts`)

- `CATALOGUE.EMPLOYEE`
- `COUPONS.HISTORY(couponId)`

### Dialogs

- `src/app/shared/components/coupon-download-dialog/` — Employee picker + Download button. Triggered from the row's Download action.
- `src/app/shared/components/coupon-history-dialog/` — Timeline UI showing every event on `coupon_request` + `coupon_download` chronologically (newest first). Vertical connector line, colored markers per action (green CREATE, blue APPEND/info, amber EDIT, red DELETE, teal DOWNLOAD). Natural-language labels: "Created", "Added more", "Edited", "Deactivated", "Downloaded". Each event: actor, delta description (e.g. "Generated 100 vouchers", "Configuration updated" for zero-delta edits), timestamp.
- `src/app/shared/components/coupon-form-dialog/*` — CREATE mode: employee picker top-left + partner picker top-right (`.field-row`, two columns). APPEND mode: employee picker only (partner inherited). Picker uses `ActingEmployeeState`; selection updates the state so it prefills the next dialog. `requestedBy` is NO LONGER a form control — it's the state selection.
- `src/app/shared/components/coupon-edit-dialog/*` — Employee picker at the top (`Requested by`). Same state pattern.
- `src/app/shared/components/coupon-delete-dialog/*` — Employee picker at the top. Confirm checkbox remains. Copy still says "Delete" (rename to "Deactivate" was deferred — see below).

### Coupons page (`src/app/pages/settings/coupons/coupons.component.*`)

- Row simplified. Meta line: `Partner · X/Y scanned`. Requester line: `{Action} by {Name} on {Date}` using `lastEvent*` fields — single piece of activity, no more "Created" or "Last downloaded" duplication.
- New per-row **History** button opens the timeline dialog.
- The USED-lock guard (`vouchersUsed > 0`) is gone — soft-delete backend removes the constraint.
- Constructor no longer preloads employees. Loading happens on `onOperatorPick(op)` via `ensureLoadedForOperator(op.operatorCompanyId)` because employees are operator-scoped.

## Design decisions worth capturing

- **Employee picker in every dialog, no page-level banner.** An initial "Test employee ID" banner at the top of the page was rejected — filters at the top must be operator/location/partner only. Attribution happens per action, in the dialog.
- **Downloads count as activity.** The row's last-event line surfaces the latest event ACROSS `coupon_request` + `coupon_download`. The history dialog is a `UNION ALL` of both tables.
- **No duplication row vs dialog.** Row shows ONE piece of activity context (the last event). Everything else — creator, download history, edit history — lives in the History dialog.
- **Natural-language labels.** `APPEND` renders as "Added more" in the UI (matches the existing "Add more" row button). `DELETE` renders as "Deactivated" in labels — though the button copy still says "Delete" pending the deferred rename.
- **Query params over headers.** DELETE and export use `?requestedBy=` / `?downloadedBy=` query params instead of custom headers to avoid touching `ms-gateway-service` CORS whitelist. Only gateway-accepted headers (Operator-Id, Location-Id, Content-Type, Authorization) are used.

## Dark/light mode

- Dropped hardcoded fallbacks like `#f9fafb` in `--color-bg-30, #f9fafb` — that variable doesn't exist. Legit tokens are `--color-bg-app/50/60/80/100`.
- History dialog uses real design-system tokens and `color-mix` for tinted badges so both themes render correctly.

## Explicitly NOT done in this iteration

- **Migration from `frontend-commerce` to `frontend-backoffice`** — postponed indefinitely. Coupons stay in `frontend-commerce` for now.
- **`Deactivate/Activate` verbiage rename + reactivation flow** — Israel asked, then deferred. Would require adding `ACTIVATE` to the DB CHECK constraint and touching the DB again post-release.

## Cross-references

- Backend: `2e24a13 feat(coupons): audit history + soft-delete + activity endpoint` on `ms-backoffice-service` `feature/coupons`, merged into `feature/staging` as `1587c86`. Backend project memory: `project_coupon_audit_ddl_2026_07_15.md`.
- DDL release: `~/Dev/Back-End/ddl/DDL_2026_07_15_COUPONS/`. Ran in DEV on 2026-07-16 (as `postgres` from AWS Dev Intranet). PROD not run yet — DBA gets the consolidated `DDL_RELEASE_COUPONS_AUDIT_HISTORY.sql`.
- Related memories: `feedback_atomic_task_workflow.md`, `reference_no_migrations.md`.
