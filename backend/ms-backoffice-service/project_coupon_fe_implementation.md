---
name: Coupon FE implementation — full page redesign detail (updated 2026-07-08 pm)
description: Files, service, page structure, dialog design, rate restrictions picker, and decisions taken for the /settings/coupons admin page in frontend-commerce. Committed on feature/coupons unpushed. Read this before modifying FE coupon code
type: project
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
Snapshot of the FE Coupons feature as **committed on `feature/coupons` (unpushed)** at end-of-day 2026-07-08. Two commits on the branch:

- `11a9ede feat(coupons): add settings page consuming backoffice bulk + export` (Iteration 2, prior session)
- `0c249e3 feat(coupons): redesign page with dialog form, table view, and rate restrictions` (this session)

TypeScript compiles clean. Browser-tested throughout the session.

## Files at their final state

**Page:**
- `src/app/pages/settings/coupons/coupons.component.{ts,html,scss}`
- `src/app/pages/settings/coupons/index.ts`

**Dialog (new):**
- `src/app/shared/components/coupon-form-dialog/coupon-form-dialog.component.{ts,html,scss}`
- `src/app/shared/components/coupon-form-dialog/index.ts`

**Types + service + paths:**
- `src/app/core/enums/coupon-mode.ts` — `CouponMode { Create = 'CREATE', Append = 'APPEND' }`
- `src/app/core/definitions/coupon.d.ts` — `CouponEntry`, `CouponBulkRequest`, `CouponBulkResult`, `RateRestriction`, `CouponSummary`, `CouponSummaries`, `AvailableRate`, `AvailableRates`
- `src/app/core/definitions/partner.d.ts` — optional `city?: string`
- `src/app/core/services/coupons-service.ts` — `createBulk`, `exportVouchers`, `getPartners`, `listSummaries`, `deleteCoupon`, `listAvailableRates`
- `src/app/core/utils/path-service-setter.ts` — `COUPONS` block with 5 entries (`BULK`, `LIST`, `DELETE(id)`, `EXPORT_VOUCHERS(id)`, `AVAILABLE_RATES`)
- `src/app/app.routes.ts` — `/settings/coupons` route
- `src/app/shared/layouts/main-layout/main-layout.component.ts` — `Coupons` menu item in SETTINGS

## Page structure

`/settings/coupons` (under Settings shell):

1. **Header** — title + subtitle.
2. **Selector row** — three `<searchable-dropdown>` in one line: **Operator**, **Location**, **Partner (filter, optional)** + a Clear button.
3. **Empty state** — before both Operator + Location are picked.
4. **Table panel** (once operator+location ready):
   - Header row with a count and **"+ New coupon batch"** button.
   - `.coupon-list` of `.coupon-row` items. Each row has:
     - `__main`: `couponCode` (mono font), partner name, and voucher counts (ACTIVE / **USED** highlighted with `--color-warning` when > 0 / CANCELLED).
     - `__actions`: **Add more** (opens dialog in add-more mode), **Download** (calls export directly), **Delete** (opens `<confirm-dialog>` with `typeToConfirm: { word: couponCode }`; button disabled with a tooltip when `vouchersUsed > 0`).

## Dialog — `<coupon-form-dialog>`

Self-contained. Uses shared `<dialog-wrapper>`. Two modes selected via `data.addMoreTarget`:

- **new** (`data.addMoreTarget` absent): full form
  - Coupon code (required, max 50)
  - Partner (searchable dropdown with `city` as sublabel)
  - How many vouchers? / Duration (min) / Price (row)
  - Valid from / Valid until (row) — `<input type="datetime-local">` → Luxon converts to ISO-with-offset in `#toIso()` before submit
  - **Restrict to specific rates (optional)** — pill-checkbox list populated from `GET /v1/coupons/available-rates` on init. Falls back to italic hint when the location has no active rates.
- **add-more** (`data.addMoreTarget: CouponSummary`): reduced form
  - Coupon code (locked, patched from the target)
  - How many more vouchers? (only field)
  - No partner picker, no rate restrictions (never mutates the coupon template)

Submit routes to `CouponsService.createBulk()` with a single-entry `coupons: [entry]`. On success emits `{ refresh: true }` on close so the page reloads the table.

## Rate restrictions picker (dialog, new mode only)

- `#loadAvailableRates()` fires in constructor when `!isAddMore()`.
- `#selectedRateKeys: WritableSignal<ReadonlySet<string>>` — key format `"{rate}|{rateVersion}"`.
- Checkbox toggle mutates the set immutably.
- `#buildRestrictions()` filters `availableRates()` by the set and maps to `{ rate, rateVersion }`.
- Empty set → `hasRateRestriction: false, rateRestrictions: []`. Non-empty → `true` + array.

## Service — `CouponsService`

All methods add `Operator-Id` + `Location-Id` headers via a private `#buildHeaders()`.

- `createBulk(op, loc, body)` → `Observable<CouponBulkResult | null>` (POST `.../coupons/bulk`)
- `exportVouchers(op, loc, couponId)` → `Observable<Blob>` (GET, `responseType: 'blob'`)
- `getPartners(op, loc)` → `Observable<Partners>` (reuses `/crm/pms/connections/partners`)
- `listSummaries(op, loc, partnerId?)` → `Observable<CouponSummaries>` (GET `.../coupons` with optional `?partnerId=`)
- `deleteCoupon(op, loc, couponId)` → `Observable<CommonResponse<void>>` (DELETE `.../coupons/{id}`)
- `listAvailableRates(op, loc)` → `Observable<AvailableRates>` (GET `.../coupons/available-rates`)

## Loader / snackbar / RxJS patterns

- Every HTTP call drives `LoaderState.show()/hide()` via `finalize()`.
- Every subscription uses `takeUntilDestroyed(this.#destroyRef)`.
- On success: `MatSnackBar` with `panelClass: 'snackbar-success'`.
- On error: extracts `err.error.message` + `err.error.errors[]` from the `ApiResponse<Void>` error envelope and concatenates.

## Design decisions taken (do not relitigate)

- **Table view + dialog form, not two-card layout** — the previous iteration had a two-card grid (create form left, export form right). Redesigned to a table-first flow so a non-technical admin can browse existing coupon batches per partner. "New coupon batch" button opens the same dialog for creation; each row's actions handle download/edit/delete.
- **"Add more" masquerades as Edit** — true field editing (price, duration, dates) is intentionally deferred until product asks. Rationale: it's the smallest useful thing to ship; when Edit needs to be built, it plugs into the same dialog with a third mode.
- **Type-to-confirm delete** — reuses shared `<confirm-dialog>` with `typeToConfirm: { word: couponCode }`. Higher friction than a simple "Are you sure?" because deletion is destructive and hard to reverse.
- **Delete button disabled with tooltip when `vouchersUsed > 0`** — the FE gate mirrors the BE `COUPON-008` rejection so the user gets the reason inline instead of a snackbar after a failed request.
- **Partner selector via `<searchable-dropdown>` reusing `/crm/pms/connections/partners`** — the query is location-scoped without PMS filter, safe to reuse today. Follow-up: build a proper `GET /v1/partners` slice on the modern envelope to decouple.
- **Rate restrictions as pill-checkboxes, not a checkbox list or multiselect** — matches design system aesthetics and communicates "chip-like tag" better than a vertical list. `:has(input:checked)` gives the active-state background.
- **Dialog loads rates on init (not the page)** — the page already carries operators/locations/partners/coupons signals; adding rates only used inside the dialog would pollute it. Dialog is scope-limited.
- **`<input type="datetime-local">` accepted** — cross-browser inconsistency accepted. Luxon guarantees the BE receives a valid `OffsetDateTime`.
- **Menu placement** in SETTINGS group (already appended in Iteration 2).

## Deliberate scope gaps

- **No true Edit UI** — waiting on product feedback. If added, extend `<coupon-form-dialog>` with a third mode.
- **`hasRateRestriction` not exposed as a separate toggle** — derived from selection count. Simplifies UX: empty picker → no restriction, any pick → restrict.
- **No dark-mode-specific overrides** beyond design tokens.
- **No inline per-field error rendering** — snackbar shows the joined `errors[]` from the BE `ApiResponse` on failure.

## Commit landed this session

`0c249e3 feat(coupons): redesign page with dialog form, table view, and rate restrictions` — 11 files, 880 insertions, 359 deletions. Contains:
- Page rewrite (`coupons.component.{ts,html,scss}`)
- New `shared/components/coupon-form-dialog/` component (4 files)
- Type additions (`coupon.d.ts`, `partner.d.ts`)
- Service additions (`coupons-service.ts`)
- Path additions (`path-service-setter.ts`)

Not pushed. Israel opens the PR manually.

## Local run

FE points at `http://localhost:8080/` (gateway) via `src/environments/environment.ts`. BE + gateway + FE all running locally. Route: `/settings/coupons` after signing in and selecting operator + location.

## Cross-links

- **BE contract**: `project_coupon_be_implementation.md`.
- **Master progress**: `project_coupon_progress_snapshot.md`.
- **Data model verified in dev**: `project_coupon_data_model.md`.
- **Release version**: `project_coupon_ship_version.md`.
- **frontend-commerce CLAUDE.md**: `/Users/israel/Dev/frontend-commerce/CLAUDE.md`.
