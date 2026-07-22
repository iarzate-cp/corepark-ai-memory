---
name: Manual coupon provisioning SQL script — the origin of the feature
description: Location and purpose of the PL/pgSQL DO block used today to bulk-create coupon templates + vouchers per location. This is the reference implementation the backend coupon feature aims to replace with an admin CRUD
type: reference
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
**Path:** `/Users/israel/Downloads/PL_COUPON_BULK_INSERT.sql` (Israel's Downloads folder — not versioned in any repo).

## What it does

PL/pgSQL `DO $$ ... $$` block that iterates a JSON array of coupon configurations and generates coupon templates + vouchers into schema `company`. Two modes per entry:

- **CREATE**: new template + N vouchers in `[sequence_min, sequence_max]`. Fails if `coupon_code` already exists at that `(operator, location)`.
- **APPEND**: add M more vouchers to an existing template, continuing from `MAX(sequence) + 1`. Ignores pricing/validity fields (uses the template's existing values).

## JSON structure the script expects (v_config)

```json
{
  "operator_company_id": 1,
  "parking_location_id": 109,
  "coupons": [
    {
      "mode": "CREATE",
      "partner_id": 68,
      "coupon_code": "ZARA-2026",
      "sequence_min": 1,
      "sequence_max": 100,
      "time_in_minutes": 300,
      "time_price": 22.00,
      "valid_from": "2026-01-20 00:00:00-06:00",
      "valid_until": "2027-12-31 23:59:59-06:00",
      "has_rate_restriction": false,
      "rate_restrictions": []
    },
    {
      "mode": "APPEND",
      "coupon_code": "ZARA-2026",
      "quantity": 100
    }
  ]
}
```

Fields per entry:
- `mode` (CREATE default, or APPEND)
- CREATE-only: `partner_id`, `sequence_min`, `sequence_max`, `time_in_minutes`, `time_price`, `valid_from`, `valid_until`, `has_rate_restriction`, `rate_restrictions[]`
- APPEND-only: either `quantity` (auto-computes range from current max) OR explicit `sequence_min`/`sequence_max`

## What the script inserts per CREATE entry

1. `company.coupon` — 1 row (template).
2. `company.coupon_type_time` — 1 row (type=1 TIME attributes).
3. `company.coupon_rate_restriction` — 0..M rows, only if `has_rate_restriction=true`.
4. `company.coupon_voucher` — N rows for sequences `sequence_min..sequence_max`, each with a `gen_random_uuid()` as `scan_code`, status `ACTIVE`.

## What the script inserts per APPEND entry

1. Updates `company.coupon.sequence_max` on the existing template.
2. `company.coupon_voucher` — M new rows continuing the sequence, `ACTIVE`.

Does NOT touch `coupon_type_time`, `coupon.valid_from/until`, nor `coupon_rate_restriction`.

## Validation the script does

- `parking_location` exists for `(operator, location)`.
- CREATE: `partner` exists for `(operator, location, partner)`.
- CREATE: `coupon_code` does not already exist at `(operator, location)`.
- APPEND: `coupon_code` exists at `(operator, location)`; either `quantity` or explicit range provided.
- `mode` is exactly `CREATE` or `APPEND`.

## Display code format for the printed voucher

`{coupon_code}-{sequence}` — e.g. `ZARA-2026-42`. In some variants of the script (verification queries at the end), the sequence is left-padded to the length of `sequence_max` (`ZARA-2026-042`). Decide with product what format ships.

## Excel export queries at the bottom

The script includes three variants of the export query (last ~50 lines). All join `coupon` + `coupon_voucher` + `coupon_type_time` + `parking_location` + `partner` + `operator_company`. Columns include: Operator, Location, Partner, Display Code, Scan Code, Time in Minutes/Hours, Price, Valid From/Until (formatted in the parking location's timezone via `AT TIME ZONE pl.parking_location_tz`).

Two scoping strategies:
- **Latest batch only:** filter by `cv.created_at >= NOW() - INTERVAL '1 hour'` (or configurable).
- **Everything for a coupon:** no time filter, filter by `coupon_uuid` or `coupon_code` instead.

The final variant uses `cv.created_at` (not `c.created_at`) so APPENDed vouchers get picked up even when the template is old.

## How to apply as reference for the backend

- The CREATE/APPEND branching in the DO block maps directly to service-layer methods (or two endpoints, depending on the final design).
- Sequence generation loop `FOR v_seq IN v_sequence_min..v_sequence_max` in Java is `IntStream.rangeClosed(min, max)` or a plain `for` loop.
- Use `GeneratedKeyHolder` for the coupon template insert (need the id back for the voucher inserts).
- Voucher inserts should use `batchUpdate` — the script inserts one-by-one which is fine for a manual DO block but wasteful in a service that runs live.

## Known caveats

- Script has a typo in the example JSON: `"rate": "Hardner"` — probably meant `"VALET"` or `"SELF"`. Doesn't crash the script (INSERTs whatever string it gets); just noise.
- Script assumes `uuid_generate_v4()` extension is available. Postgres modern default uses `gen_random_uuid()` from `pgcrypto`. Verify which is loaded in dev.
