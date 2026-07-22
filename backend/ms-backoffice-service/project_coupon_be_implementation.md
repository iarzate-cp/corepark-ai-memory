---
name: Coupon BE implementation — full slice detail (updated 2026-07-08 pm)
description: Files, endpoint URLs, ErrorCodes, DAO/service structure, and design decisions for the com.corepark.backoffice.coupon slice. Full CRUD shipped. Read this before modifying BE coupon code
type: project
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
Snapshot of the `com.corepark.backoffice.coupon` slice as it stands **committed on `feature/coupons`** (unpushed) at end-of-day 2026-07-08. **All tests green: 58/58** (21 service + 8 controller + 29 DAO).

## Endpoints (all under `@RequestMapping("/v1/coupons")`)

1. **`POST /v1/coupons/bulk`** — bulk CREATE/APPEND. `@Transactional` (whole batch rolls back if any entry fails).
   - Body: `CouponBulkRequest` = `{ coupons: List<CouponEntryRequest> }`.
   - Response: `ApiResponse<CouponBulkResponse>` = `{ couponsCreated, vouchersCreated, vouchersAppended, createdCouponIds[] }`.
2. **`GET /v1/coupons`** — summaries at a location, optional `?partnerId=` filter.
   - Response: `ApiResponse<List<CouponSummary>>` with voucher counts by status, timing/price, validity window, `createdAt`.
3. **`DELETE /v1/coupons/{couponId}`** — hard-delete with USED-lock.
   - 409 `COUPON-008` when any voucher status is `USED`.
   - Otherwise cascades across the 4 tables in-order inside one `@Transactional` boundary: `coupon_voucher` → `coupon_rate_restriction` → `coupon_type_time` → `coupon`. **No FKs exist**, so cascade is manual.
4. **`GET /v1/coupons/{couponId}/vouchers/export`** — Excel export.
   - `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
   - `Content-Disposition: attachment; filename="coupon-{id}-vouchers.xlsx"`
5. **`GET /v1/coupons/available-rates`** — active rates at the location for the rate-restriction picker.
   - Response: `ApiResponse<List<AvailableRate>>` = `[{ rate, rateVersion, rateTypeId }]` from `company.parking_location_rate WHERE active = TRUE`.

Headers required on every endpoint (except export/download): `Operator-Id`, `Location-Id` (both `int` primitives → Spring auto-400s on missing).

## ErrorCode entries in `com.corepark.common.exception.ErrorCode`

- `COUPON_INVALID_LOCATION`         → 400 · `COUPON-001`
- `COUPON_INVALID_PARTNER`          → 400 · `COUPON-002`
- `COUPON_CODE_ALREADY_EXISTS`      → 409 · `COUPON-003`
- `COUPON_CODE_NOT_FOUND`           → 404 · `COUPON-004` (APPEND + export + delete + list-by-id)
- `COUPON_INVALID_MODE`             → 400 · `COUPON-005`
- `COUPON_APPEND_MISSING_RANGE`     → 400 · `COUPON-006`
- `COUPON_EXPORT_FAILED`            → 500 · `COUPON-007`
- `COUPON_HAS_USED_VOUCHERS`        → 409 · `COUPON-008` (delete rejection)

Reserved for a future Edit iteration (not added yet): `COUPON_PARTNER_IMMUTABLE`, `COUPON_LOCKED_BY_USED_VOUCHERS` (both 409).

## Slice structure

```
com.corepark.backoffice.coupon/
├── bean/
│   ├── enumerators/{CouponMode, CouponVoucherStatus}.java
│   ├── request/{CouponBulkRequest, CouponEntryRequest, RateRestrictionRequest}.java
│   ├── response/{CouponBulkResponse, CouponSummary, AvailableRate}.java
│   └── row/{CouponRow, CouponVoucherExportRow}.java
├── controller/{CouponController, CouponApiExamples}.java
├── dao/{CouponDao, impl/CouponDaoImpl}.java
└── service/{CouponService, impl/{CouponServiceImpl, CouponExcelGenerator}}.java

src/test/java/com/corepark/backoffice/coupon/
├── controller/CouponControllerTest.java   (new — 8 tests)
├── dao/impl/CouponDaoImplTest.java        (new — 29 tests)
└── service/impl/CouponServiceImplTest.java (extended — 21 tests total)
```

## DAO method breakdown (one op per method per CLAUDE.md)

- `existsLocation(operator, location) → boolean`
- `existsPartner(operator, location, partner) → boolean`
- `existsCouponCode(operator, location, code) → boolean` (CREATE guard)
- `findCouponByCode(operator, location, code) → Optional<CouponRow>` (APPEND lookup — id + `COALESCE(MAX(cv.sequence), 0)`)
- `insertCoupon(...) → long` (via `GeneratedKeyHolder`)
- `insertCouponTypeTime(couponId, timeInMinutes, timePrice) → int`
- `insertCouponRateRestrictions(couponId, operator, location, restrictions) → int` (batchUpdate; 0 if list null/empty)
- `updateCouponSequenceMax(couponId, newSequenceMax) → int`
- `insertVouchers(couponId, sequenceMin, sequenceMax) → int` — batchUpdate; each voucher gets `UUID.randomUUID().toString()` as `scan_code`, `ACTIVE` status
- `findVouchersForExport(operator, location, couponId) → List<CouponVoucherExportRow>`
- `findSummariesByLocation(operator, location, partnerId) → List<CouponSummary>` — nullable `partnerId` bound with `Types.INTEGER` so `(:partnerId IS NULL OR c.partner_id = :partnerId)` works
- `findSummaryById(operator, location, couponId) → Optional<CouponSummary>` — same SELECT as above, filtered by id (used by delete)
- `deleteVouchersByCouponId(couponId) → int`
- `deleteRateRestrictionsByCouponId(couponId) → int`
- `deleteCouponTypeTimeByCouponId(couponId) → int`
- `deleteCouponById(couponId) → int`
- `findAvailableRates(operator, location) → List<AvailableRate>` — `SELECT DISTINCT` on `parking_location_rate WHERE active = TRUE`

## Service logic

### processBulk (CREATE / APPEND) — unchanged since Iteration 1
1. Validate `existsLocation`.
2. For each entry: dispatch on `getModeOrDefault()`.
   - CREATE: field presence + range + date sanity → `existsPartner` → `existsCouponCode` → insert coupon + type-time + optional rate restrictions + vouchers.
   - APPEND: require `quantity` or explicit range → `findCouponByCode` → compute new range → `updateCouponSequenceMax` → `insertVouchers`.
3. Return counters + `createdCouponIds`.

### listSummaries (new today)
1. Validate `existsLocation`.
2. If `partnerId != null`, validate `existsPartner`.
3. Delegate to `findSummariesByLocation`.

### deleteCoupon (new today)
1. `findSummaryById` → `COUPON_CODE_NOT_FOUND` if empty.
2. If `summary.vouchersUsed > 0` → `COUPON_HAS_USED_VOUCHERS`.
3. Otherwise cascade-delete in the 4-table order.

### listAvailableRates (new today)
1. Validate `existsLocation`.
2. Delegate to `findAvailableRates`.

### exportVouchers — unchanged
1. `findVouchersForExport` → `COUPON_CODE_NOT_FOUND` if empty.
2. Delegate to `CouponExcelGenerator.build(rows)`; wrap any exception as `COUPON_EXPORT_FAILED`.

## Design decisions (do not relitigate)

- **Delete is hard-delete, not soft-delete** — chosen because the DBA is unavailable for DDL and Israel asked for "no DB changes". USED-lock preserves history. Revisit for soft-delete when a `deactivated_at` column can be added.
- **Cascade is manual across 4 tables** — coupon schema has zero FKs. Delete order chosen so child rows go before parents.
- **`/v1/coupons/available-rates` is a slice-local endpoint**, not a reuse of `/rate/get-rates-catalogue`. Reason: the legacy endpoint is on the old `Response` + `MessageCode` envelope; keeping the coupon slice on `ApiResponse<T>` end-to-end avoids the FE juggling two envelopes.
- **`Operator-Id` + `Location-Id` are `int` primitives** — not wrapped in a `RequestContext`. Spring auto-400s on missing headers. Same rationale as Iteration 1.
- **`AvailableRate` includes `rateTypeId`** even though the FE picker doesn't display it — future use if we need to distinguish types.
- **`ApiResponse.noContent()` inside `ResponseEntity.ok(...)`** on `deleteCoupon` returns HTTP 200 with body `{ status: 204, code: "NO-CONTENT" }`. Known inconsistency vs the survey pattern (which uses `ResponseEntity.status(response.getStatus())`) — kept as-is to match the rest of this controller's style. If it becomes a problem, one-line change.
- **`ResponseEntity.ok(ApiResponse.success(data))`** everywhere else in the controller (not `ResponseEntity.status(response.getStatus())`) — matches Iteration 1's style; consistent within the slice.
- **`AvailableRate` uses `BeanPropertyRowMapper<AvailableRate>`** — column aliases in the SQL (`AS rate, AS rateVersion, AS rateTypeId`) map straight to fields. No custom RowMapper needed.

## Testing details

### `CouponServiceImplTest` (21 tests)
- Bulk (CREATE happy + rate restrictions inserted + APPEND from `MAX(sequence)` + inverted range + invalid location/partner/duplicate code + APPEND without range + APPEND unknown code + mode defaults to CREATE + …).
- List summaries (no filter + with filter + invalid location + invalid partner).
- Delete (happy cascade + USED > 0 throws + missing throws NOT_FOUND).
- Available rates (returns list + invalid location throws).
- Export (happy XLSX bytes + empty throws NOT_FOUND).

### `CouponControllerTest` (8 tests)
- All happy paths via `mock(CouponService.class)` + real controller.
- BindingResult with 1 and 2 field errors → `VALIDATION-FAILED` envelope + concatenated `field: message` strings.
- Export headers (`Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `Content-Disposition: attachment; filename="coupon-42-vouchers.xlsx"`).

### `CouponDaoImplTest` (29 tests)
- Each `exists*` returns null → false.
- `findCouponByCode` / `findSummaryById` → present + empty.
- `insertCoupon` uses `doAnswer(...)` to populate the `GeneratedKeyHolder.keyList` and asserts the extracted id.
- `insertCouponRateRestrictions` returns 0 on null/empty.
- Batch inserts (`insertVouchers`, `insertCouponRateRestrictions`) verify the batch length via `ArgumentCaptor<SqlParameterSource[]>`.
- `findSummariesByLocation` uses `ArgumentCaptor` to verify `params.getSqlType("partnerId") == Types.INTEGER` for the nullable filter.

## Local commands

```bash
# from ms-backoffice-service/
./mvnw test -Dtest='CouponControllerTest,CouponDaoImplTest,CouponServiceImplTest'   # runs the 58 coupon tests
./mvnw -q -DskipTests compile                                                        # sanity compile
```

## Commits on `feature/coupons` (this session)

- `1feb33c feat(coupon): add list, delete, and available-rates endpoints` — 10 files, 524 insertions.
- `12bf10f test(coupon): add controller test coverage` — 1 file, 167 insertions.
- `dd4608d test(coupon): add DAO test coverage` — 1 file, 395 insertions.

On top of the prior `07d0106 feat(coupon): add bulk provisioning and Excel export slice` + `9ee0376 Merge branch 'main' into feature/coupons`.

Not pushed. Israel opens PRs manually.
