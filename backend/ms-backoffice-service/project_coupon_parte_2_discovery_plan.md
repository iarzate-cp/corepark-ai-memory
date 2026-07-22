---
name: Coupon feature — Parte 2 discovery plan (SUPERSEDED 2026-07-08 pm)
description: Original design for the "list existing coupons" gap. Superseded by the table-first redesign shipped 2026-07-08 pm (see project_coupon_progress_snapshot.md). Kept for design history only — do NOT resurrect
type: project
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
**⚠️ SUPERSEDED 2026-07-08 pm.** The gap this plan closed is now shipped as part of the table-first redesign (see `project_coupon_progress_snapshot.md` + `project_coupon_fe_implementation.md`). This document is retained only to preserve the earlier design thinking. Do NOT treat any of the details below as current guidance.

**What actually shipped instead**: `GET /v1/coupons` (identical to Parte 2's proposal) but the FE became a full-page table with row actions (Add more / Download / Delete) and an extracted `<coupon-form-dialog>`, not a "discovery panel" alongside the create form. Rate restrictions were also added in the same wave.

## The gap this closes

After the browser smoke-test of Iteration 1+2 succeeded (Coupon id=34, 100 vouchers, Excel exported correctly), Israel asked: *"cómo sabemos que un partner ya tiene cupones?"*

Right now: nowhere. The admin has to remember, query the DB directly, or attempt CREATE and read the `COUPON_CODE_ALREADY_EXISTS` (409) error. None acceptable for production admin use.

Parte 2 fixes it: **when the admin picks a partner, show their existing coupons with quick actions (APPEND / Download)**.

## Non-obvious architectural bonus

The endpoint's response includes `vouchersUsed`. This is **exactly** the field Iteration 3 (Edit) needs to enforce USED-lock. Building Parte 2 now means Iteration 3's FE has the flag available without another roundtrip. Ordering (`2 → 3`) is correct; don't skip Parte 2 to jump to Edit.

## BE — new endpoint

### `GET /v1/coupons`

- **Headers**: `Operator-Id`, `Location-Id` (both `int` primitives, standard pattern).
- **Query params**:
  - `partnerId` (optional, `int`) — filters to a specific partner. Omit to return all coupons at the location.
- **Response**: `ApiResponse<List<CouponSummary>>`.
- **Empty list** if none match — do NOT throw `COUPON_CODE_NOT_FOUND` here (empty is a valid state).

### `CouponSummary` response DTO

```java
package com.corepark.backoffice.coupon.bean.response;

// under bean/response/ per the slice's existing structure
public class CouponSummary {
    private long           id;
    private String         couponCode;
    private int            partnerId;
    private String         partnerName;
    private long           sequenceMin;
    private long           sequenceMax;
    private int            vouchersActive;      // status='ACTIVE' count
    private int            vouchersUsed;        // status='USED' count — feeds Iteration 3 USED-lock UI
    private int            vouchersCancelled;   // status='CANCELLED' count
    private int            timeInMinutes;
    private BigDecimal     timePrice;
    private OffsetDateTime validFrom;
    private OffsetDateTime validUntil;
    private OffsetDateTime createdAt;
}
```

Lombok: `@Getter @Setter @NoArgsConstructor` (row bean convention).

### DAO — one method

`findSummariesByLocation(operator, location, partnerId /* nullable */) → List<CouponSummary>`

SQL (a single query with LEFT JOIN aggregates):

```sql
SELECT
  c.id                AS id,
  c.coupon_code       AS couponCode,
  c.partner_id        AS partnerId,
  p.name              AS partnerName,
  c.sequence_min      AS sequenceMin,
  c.sequence_max      AS sequenceMax,
  COALESCE(SUM(CASE WHEN cv.status = 'ACTIVE'    THEN 1 ELSE 0 END), 0) AS vouchersActive,
  COALESCE(SUM(CASE WHEN cv.status = 'USED'      THEN 1 ELSE 0 END), 0) AS vouchersUsed,
  COALESCE(SUM(CASE WHEN cv.status = 'CANCELLED' THEN 1 ELSE 0 END), 0) AS vouchersCancelled,
  ctt.time_in_minutes AS timeInMinutes,
  ctt.time_price      AS timePrice,
  c.valid_from        AS validFrom,
  c.valid_until       AS validUntil,
  c.created_at        AS createdAt
FROM company.coupon c
JOIN company.coupon_type_time ctt
  ON ctt.coupon_id = c.id
JOIN company.partner p
  ON p.operator_company_id = c.operator_company_id
 AND p.parking_location_id = c.parking_location_id
 AND p.partner_id          = c.partner_id
LEFT JOIN company.coupon_voucher cv
  ON cv.coupon_id = c.id
WHERE c.operator_company_id = :operatorCompanyId
  AND c.parking_location_id = :parkingLocationId
  AND (:partnerId IS NULL OR c.partner_id = :partnerId)
GROUP BY c.id, ctt.coupon_id, p.partner_id, p.operator_company_id, p.parking_location_id
ORDER BY c.created_at DESC
```

Binding: `parameters.addValue("partnerId", partnerId, Types.INTEGER)` when null-guarded. Follow the CLAUDE.md rule `(:param IS NULL OR col = :param)` with explicit SQL type.

Row mapper: `BeanPropertyRowMapper<>(CouponSummary.class)` — all aliases match camelCase properties.

### Service — one method

`listSummaries(operatorCompanyId, parkingLocationId, partnerId /* nullable */) → List<CouponSummary>`

- Validates location exists (throws `COUPON_INVALID_LOCATION` if not).
- If `partnerId` provided, validates partner exists at that location (throws `COUPON_INVALID_PARTNER` if not).
- Delegates to DAO.
- No new `@Transactional` needed; read-only by default under Spring.

### Controller — new mapping

Add to `CouponController` (already `@RequestMapping("/v1/coupons")`):

```java
@Operation(
    summary = "List coupons at a location, optionally filtered by partner",
    ...
)
@GetMapping
public ResponseEntity<ApiResponse<List<CouponSummary>>> listSummaries(
    @RequestHeader("Operator-Id") final int operatorCompanyId,
    @RequestHeader("Location-Id") final int parkingLocationId,
    @RequestParam(value = "partnerId", required = false) final Integer partnerId) {
    List<CouponSummary> data = couponService.listSummaries(operatorCompanyId, parkingLocationId, partnerId);
    return ResponseEntity.ok(ApiResponse.success(data));
}
```

Add `LIST_RESPONSE` example to `CouponApiExamples`.

### Tests

Extend `CouponServiceImplTest`:
- Happy path with 3 coupons at location, no partner filter → 3 returned.
- Same with `partnerId` filter → subset.
- Invalid location → `COUPON_INVALID_LOCATION`.
- Invalid partner → `COUPON_INVALID_PARTNER`.
- Empty result → returns empty list (no throw).

## FE — component + service changes

### `path-service-setter.ts`

Extend the `COUPONS` block:

```ts
COUPONS: {
  BULK: 'backoffice/v1/coupons/bulk',
  LIST: 'backoffice/v1/coupons',   // NEW
  EXPORT_VOUCHERS: (couponId: number) => `backoffice/v1/coupons/${couponId}/vouchers/export`,
}
```

### `coupon.d.ts`

Add:

```ts
export interface CouponSummary {
  readonly id: number
  readonly couponCode: string
  readonly partnerId: number
  readonly partnerName: string
  readonly sequenceMin: number
  readonly sequenceMax: number
  readonly vouchersActive: number
  readonly vouchersUsed: number
  readonly vouchersCancelled: number
  readonly timeInMinutes: number
  readonly timePrice: number
  readonly validFrom: string
  readonly validUntil: string
  readonly createdAt: string
}

export type CouponSummaries = readonly CouponSummary[]
```

### `coupons-service.ts`

New method:

```ts
listSummaries(operatorCompanyId: number, parkingLocationId: number, partnerId?: number) {
  const url = pathServiceSetter(API_PATHS.COUPONS.LIST)
  const headers = this.#buildHeaders(operatorCompanyId, parkingLocationId)
  const params = partnerId !== undefined ? new HttpParams().set('partnerId', String(partnerId)) : undefined
  return this.#http
    .get<CommonResponse<CouponSummaries>>(url, { headers, params })
    .pipe(map((r): CouponSummaries => r?.data ?? []))
}
```

### `coupons.component.ts`

- New signal: `#coupons = signal<CouponSummaries>([])`.
- New computed: `filteredCoupons` (client-side filter by selected partner if any; server can also do it, choice of consistency).
- New method: `#loadCoupons(op, loc, partnerId?)` — drives loader, fills the signal.
- Call `#loadCoupons` in:
  - `onLocationPick` (all coupons at location).
  - `onPartnerPick` (re-filter or re-fetch scoped to partner).
- New handlers:
  - `onAppendClick(summary)` — pre-fills the form: `mode=APPEND`, `couponCode=summary.couponCode`. Scrolls to form. User just adds `quantity`.
  - `onDownloadClick(summary)` — directly calls `exportVouchers(op, loc, summary.id)` without needing to type the ID.

### `coupons.component.html`

**New section** below the partner selector, shown only when a partner is picked and their coupon list is non-empty:

```html
@if (selectedPartner() && filteredCoupons().length) {
  <div class="existing-coupons">
    <header class="existing-coupons__header">
      <h3 class="existing-coupons__title">Existing coupons for {{ partnerDisplay() }}</h3>
      <span class="existing-coupons__count">{{ filteredCoupons().length }}</span>
    </header>
    <ul class="existing-coupons__list">
      @for (summary of filteredCoupons(); track summary.id) {
        <li class="coupon-row">
          <div class="coupon-row__meta">
            <span class="coupon-row__code">{{ summary.couponCode }}</span>
            <span class="coupon-row__id">#{{ summary.id }}</span>
            <span class="coupon-row__vouchers">
              {{ summary.vouchersActive + summary.vouchersUsed + summary.vouchersCancelled }} vouchers
              @if (summary.vouchersUsed > 0) {
                · <span class="coupon-row__used">{{ summary.vouchersUsed }} used</span>
              }
            </span>
          </div>
          <div class="coupon-row__actions">
            <button type="button" class="ui-button ui-button--secondary" (click)="onAppendClick(summary)">Append</button>
            <button type="button" class="ui-button ui-button--secondary" (click)="onDownloadClick(summary)">Download Excel</button>
          </div>
        </li>
      }
    </ul>
  </div>
}
```

**Export card** on the right: replace the `<input type="number">` for `couponId` with a `<searchable-dropdown>` fed by `filteredCoupons` (or the full list at the location — TBD; probably full list, since export by ID doesn't need partner filter). Each item's label: `{couponCode} #{id}`.

## Open UX questions (parkéd — decide during implementation)

- **Where does `onAppendClick` navigate the eye?** Options: (A) scroll to top of the CREATE form, (B) do nothing (assume the form is already visible on the same viewport). Test both; if the form is below the fold, option A.
- **Confirm before Download button** — probably no, downloads should be one-click.
- **Refresh after bulk create** — after a successful `createBulk`, should we auto-refresh `#coupons` so the new one appears in the list? Probably yes; add a call to `#loadCoupons(...)` in the `onSubmit` success handler.
- **Pagination** — how many coupons can a location have? If dozens: fine, one query, one panel. If hundreds: need paging. For MVP assume dozens.

## What Parte 2 does NOT include

- Editing individual coupons (that's Iteration 3).
- Deleting coupons (that's Iteration 4).
- Bulk actions on the list (delete multiple, cancel vouchers, etc.).
- Search/filter within the list (add if the list grows > 20 items).

## Estimated scope

- **BE**: ~1h — 1 new bean, 1 DAO method (with the aggregate query above), 1 service method with 2 validations, 1 controller endpoint with springdoc-openapi + `CouponApiExamples` entry, extend `CouponServiceImplTest` with ~4 test cases.
- **FE**: ~1h — path constant, `CouponSummary` type, service method, 2 new signals + 2 new handlers in the component, list template + basic styles, replace export card input with dropdown, ensure the loader/snackbar patterns.

**Total ~2h.** Small enough to fit in one focused block.

## When to start

Israel to give the go. Right now (2026-07-08 end-of-session) is the natural stopping point after browser smoke-test + Parte 1 done. Next session: pick this up, code BE first, verify with Postman, then wire the FE.
