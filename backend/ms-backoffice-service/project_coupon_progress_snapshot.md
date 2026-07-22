---
name: Coupon feature — progress snapshot 2026-07-08 (end of day — full CRUD shipped on branches, unpushed)
description: Master snapshot. Full coupon CRUD (bulk, list, delete-when-no-USED, export, available-rates) shipped end-to-end + rate restrictions UI. 4 atomic commits landed on feature/coupons across both repos. Not pushed. Target release is frontend-commerce v3.6.0
type: project
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
**Status as of 2026-07-08 end-of-day: full coupon CRUD + rate restrictions shipped on `feature/coupons` in both repos. Not pushed.** 4 atomic commits landed today. Browser-tested green throughout. Target release: frontend-commerce v3.6.0 (see `project_coupon_ship_version.md`).

## Working-tree state right now

- **`ms-backoffice-service`** on `feature/coupons`, working tree clean, 3 new commits ahead of the previous snapshot:
  - `dd4608d test(coupon): add DAO test coverage` — new `CouponDaoImplTest.java`, 29 tests
  - `12bf10f test(coupon): add controller test coverage` — new `CouponControllerTest.java`, 8 tests
  - `1feb33c feat(coupon): add list, delete, and available-rates endpoints`
  - Prior HEAD: `9ee0376 Merge branch 'main' into feature/coupons` on top of `07d0106 feat(coupon): add bulk provisioning and Excel export slice`
  - **Total BE tests: 58/58 green** (21 service + 8 controller + 29 DAO).
- **`frontend-commerce`** on `feature/coupons`, working tree clean, 1 new commit ahead:
  - `0c249e3 feat(coupons): redesign page with dialog form, table view, and rate restrictions`
  - Prior HEAD: `f28717c Merge branch 'main' into feature/coupons` on top of `11a9ede feat(coupons): add settings page consuming backoffice bulk + export`

Neither branch has been pushed. Israel opens PRs manually on GitHub.

## What shipped today (2026-07-08)

### BE — three new endpoints on `com.corepark.backoffice.coupon`

1. **`GET /v1/coupons`** — list summaries at a location, optional `?partnerId=` filter. Returns `ApiResponse<List<CouponSummary>>` where each entry has voucher counts (ACTIVE / USED / CANCELLED), timing/price, validity window, and `createdAt`. The `vouchersUsed` field drives the FE delete-button lock.
2. **`DELETE /v1/coupons/{couponId}`** — hard-delete with USED-lock. Rejects with 409 `COUPON-008` when the coupon has any USED voucher; otherwise cascades in-order across the 4 tables (there are still no FKs on the coupon schema) inside a single `@Transactional` boundary: `coupon_voucher` → `coupon_rate_restriction` → `coupon_type_time` → `coupon`. Chosen over soft-delete because the DBA is unavailable to add DDL right now.
3. **`GET /v1/coupons/available-rates`** — active-rates picker for the coupon dialog. Returns `ApiResponse<List<AvailableRate>>` = `[{ rate, rateVersion, rateTypeId }]`. Query is `SELECT DISTINCT ... FROM company.parking_location_rate WHERE ... AND active = TRUE`. Reason for a dedicated endpoint in the coupon slice (not reusing the legacy `/rate/get-rates-catalogue`): keeps the FE on one envelope (modern `ApiResponse<T>`) and the slice self-contained. Follow-up if needed: dedicated `/v1/rates` slice on the modern envelope.

New ErrorCode: `COUPON_HAS_USED_VOUCHERS` (409, `COUPON-008`).
New DTOs: `bean/response/CouponSummary.java`, `bean/response/AvailableRate.java`.
Test files added: `test/coupon/controller/CouponControllerTest.java` (8 tests), `test/coupon/dao/impl/CouponDaoImplTest.java` (29 tests).

### FE — full page redesign at `/settings/coupons`

- **Selector row**: Operator + Location + optional Partner filter (three `<searchable-dropdown>` in one line).
- **Table view**: coupons in a card list showing code, partner, and vouchers-by-status (ACTIVE / USED highlighted with `--color-warning` when >0 / CANCELLED).
- **Row actions**: **Add more** (opens dialog in add-more mode), **Download** (calls export directly), **Delete** (opens shared `<confirm-dialog>` with `typeToConfirm: { word: couponCode }`; disabled with tooltip when `vouchersUsed > 0`).
- **Dialog extraction**: form moved out of the page into `shared/components/coupon-form-dialog/`. Two modes: **new** (full form with partner dropdown, quantity/duration/price, dates, rate-restrictions picker) and **add-more** (only quantity, couponCode locked, no rate picker). Emits `{ refresh: true }` on close so the page reloads the table.
- **Rate restrictions picker**: pill-style checkboxes below the dates row (new mode only). Populated from `GET /v1/coupons/available-rates`. Empty selection → `hasRateRestriction: false`, `rateRestrictions: []`. Any selection → `true` + array of `{ rate, rateVersion }`. Fallback text when the location has no active rates.

### Type + service additions on FE

- `coupon.d.ts` — `CouponSummary`, `CouponSummaries`, `AvailableRate`, `AvailableRates` (plus existing `RateRestriction`).
- `partner.d.ts` — optional `city?: string` (used as sublabel in partner dropdown).
- `path-service-setter.ts` — `COUPONS.LIST`, `COUPONS.DELETE(id)`, `COUPONS.AVAILABLE_RATES`.
- `CouponsService` — `listSummaries()`, `deleteCoupon()`, `listAvailableRates()`, `getPartners()`.

## Iteration plan — current status

| # | Scope | Status |
|---|---|---|
| 1 | BE: `POST /v1/coupons/bulk` + `GET /{id}/vouchers/export` | ✅ Committed as `07d0106` (prior session). |
| 2 | FE: initial `/settings/coupons` page | ✅ Committed as `11a9ede` (prior session). |
| 2.5 | BE: `GET /v1/coupons` list + FE table view + partner filter | ✅ Shipped today, part of commit `1feb33c` (BE) + `0c249e3` (FE). |
| 3 | Edit (BE + FE) | 🟡 **Explicitly deferred until product feedback.** "Add more" masquerades as edit for now. |
| 4 | Delete (BE + FE) | ✅ Shipped today as hard-delete when `vouchersUsed = 0`, part of `1feb33c` + `0c249e3`. |
| 5 | Rate restrictions (BE + FE) | ✅ Shipped today. `GET /v1/coupons/available-rates` + multi-select picker in dialog. |

## Delete design — decided today, hard-delete-when-no-USED (Option A)

- Reject with 409 `COUPON-008` if any `coupon_voucher.status = 'USED'` — preserves redemption history.
- Otherwise cascade-delete in-order across the 4 tables inside one `@Transactional` boundary. Order matters (no FKs → no cascade to lean on): `coupon_voucher` → `coupon_rate_restriction` → `coupon_type_time` → `coupon`.
- **Why hard-delete not soft-delete**: DBA unavailable for adding a `deactivated_at` column; Israel explicitly asked for "no DB changes". Trade-off documented — revisit with soft-delete when the DBA is around.

## Tests

- **BE**: 58/58 green.
  - `CouponServiceImplTest`: 21 tests (bulk + list + delete + rates + validation + edge cases).
  - `CouponControllerTest`: 8 tests (5 happy paths + 2 BindingResult flows + XLSX headers).
  - `CouponDaoImplTest`: 29 tests (each DAO method, `ArgumentCaptor` verification of `Types.INTEGER` for the nullable `partnerId` filter).
- **FE**: TypeScript compiles clean (unrelated pre-existing errors in `customers`, `auth`, `e2e` — not touched by this feature).

## Open items — parked but tracked

- **Push + PRs to `main`** — Israel opens PRs manually.
- **Edit (true field editing when `vouchersUsed = 0`)** — waiting on product feedback. Frozen rules from the earlier session (partner immutable, USED-lock scope) still stand when this comes back.
- **QA edge cases** — APPEND to a code with existing USED vouchers (should succeed, USED preserved), delete of a coupon with only CANCELLED, partner filter with 0 results, download of a coupon with USED.
- **Coordinated deploy story** — FE now depends on the three new BE endpoints. BE must land in dev before FE PR merges without breaking `/settings/coupons`.
- **`<operator-location-selector>` shared component** — ~9 duplicated pages. Not urgent.
- **Dedicated `/v1/partners` slice** — decouple from PMS reuse. Not urgent.
- **Voucher lifecycle confirmation with ms-valet-service** — who transitions ACTIVE → USED, does that path emit an event we need to hear?

## Release target

**frontend-commerce v3.6.0.** See `project_coupon_ship_version.md`. When preparing the release PR, bump `package.json` accordingly.

## To resume next session

```bash
# ms-backoffice-service/
git checkout feature/coupons
git log --oneline -5   # HEAD should be dd4608d test(coupon): add DAO test coverage

# frontend-commerce/
git checkout feature/coupons
git log --oneline -3   # HEAD should be 0c249e3 feat(coupons): redesign page ...

# Next natural step: push both branches and open the two PRs against main.
```

## Cross-links

- **BE implementation detail**: `project_coupon_be_implementation.md` (updated 2026-07-08 pm).
- **FE implementation detail**: `project_coupon_fe_implementation.md` (updated 2026-07-08 pm).
- **Data model** (verified in dev): `project_coupon_data_model.md`.
- **Release version**: `project_coupon_ship_version.md`.
- **Origin script** (obsolete after this feature ships): `reference_coupon_sql_script.md`.
