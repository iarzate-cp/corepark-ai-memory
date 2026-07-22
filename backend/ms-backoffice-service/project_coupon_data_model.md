---
name: Coupon data model — verified against dev 2026-07-08
description: Four tables in schema `company` supporting coupon templates + individual vouchers. Verified directly against coreparkdev via information_schema + pg_indexes on 2026-07-08. Includes columns, indexes, absence of FKs, and partner scoping
type: project
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
**Verified on 2026-07-08 against coreparkdev.** All findings below come from live `information_schema.columns` / `pg_indexes` queries — not inferred from the SQL script.

## Tables (schema `company`)

### `company.coupon` — the template (13 columns)

| Column | Type | Nullable | Default |
|---|---|---|---|
| `id` | bigint | NO | `nextval('company.coupon_id_seq'::regclass)` — bigserial |
| `uuid` | uuid | NO | `uuid_generate_v4()` — DB generates |
| `operator_company_id` | integer | NO | — |
| `parking_location_id` | integer | NO | — |
| `partner_id` | integer | NO | — |
| `coupon_code` | varchar | NO | — |
| `sequence_min` | bigint | NO | — |
| `sequence_max` | bigint | NO | — |
| `coupon_type_id` | integer | NO | — |
| `has_rate_restriction` | boolean | NO | — |
| `valid_from` | timestamptz | NO | — |
| `valid_until` | timestamptz | NO | — |
| `created_at` | timestamptz | NO | `now()` |

### `company.coupon_type_time` — 1:1 with coupon for TIME type (4 columns)

| Column | Type | Nullable | Default |
|---|---|---|---|
| `coupon_id` | bigint | NO | — |
| `coupon_type_id` | integer | NO | **`1`** — DB default (TIME) |
| `time_in_minutes` | integer | NO | — |
| `time_price` | numeric | NO | — |

### `company.coupon_voucher` — issued vouchers (6 columns)

| Column | Type | Nullable | Default |
|---|---|---|---|
| `id` | bigint | NO | `nextval('company.coupon_voucher_id_seq'::regclass)` — bigserial |
| `coupon_id` | bigint | NO | — |
| `sequence` | bigint | NO | — |
| `scan_code` | **varchar** | NO | — |
| `status` | varchar | NO | — |
| `created_at` | timestamptz | NO | `now()` |

**Important**: `scan_code` is **varchar**, NOT `uuid`. The script casts `uuid_generate_v4()::TEXT`. Bind as `String` in Java (`UUID.randomUUID().toString()`).

### `company.coupon_rate_restriction` — 0..M per coupon, only when `has_rate_restriction=true` (5 columns)

| Column | Type | Nullable |
|---|---|---|
| `coupon_id` | bigint | NO |
| `operator_company_id` | integer | NO |
| `parking_location_id` | integer | NO |
| `rate` | varchar | NO |
| `rate_version` | timestamptz | NO |

## Indexes (from `pg_indexes`)

### `company.coupon`
- `pk_coupon` — UNIQUE on `(id)`
- `uq_coupon_uuid` — UNIQUE on `(uuid)`
- **`uq_coupon_coupon_code`** — UNIQUE on `(coupon_code, parking_location_id, operator_company_id)` → DB-level guarantee of code uniqueness per location
- `ix_coupon_valid_dates` — btree on `(valid_from, valid_until)`
- `ix_coupon_partner` — btree on `(operator_company_id, parking_location_id, partner_id)`
- `ix_coupon_type` — btree on `(coupon_type_id)`

### `company.coupon_voucher`
- `pk_coupon_voucher` — UNIQUE on `(id)`
- **`uq_coupon_voucher_coupon_sequence`** — UNIQUE on `(coupon_id, sequence)` → prevents duplicate sequence within a coupon
- **`uq_coupon_voucher_scan_code`** — UNIQUE on `(scan_code)` → global uniqueness of scan codes
- `ix_coupon_voucher_status` — btree on `(status)`
- `ix_coupon_voucher_coupon` — btree on `(coupon_id)`

## Foreign Keys — none

**Zero FKs** on all 4 coupon* tables. Verified via `information_schema.table_constraints`. Implications:
- Referential integrity is app-responsibility. The service must explicitly validate `operator/location`, `partner`, and coupon existence — exactly what the script does with `EXISTS` checks.
- No cascade delete: Iteration 4 (Delete) must delete children in explicit order within a transaction: `coupon_voucher` → `coupon_rate_restriction` → `coupon_type_time` → `coupon`.
- Orphan risk: if a `partner` is deleted elsewhere, coupon rows survive pointing at nothing. Out of scope for this feature.

## Partner scoping — per-location, confirmed

`company.partner` has `parking_location_id` (NOT NULL) as column 15. A partner belongs to ONE parking location.

Impact:
- Coupon endpoints validate partner with `WHERE operator_company_id = ? AND parking_location_id = ? AND partner_id = ?`.
- FE partner selector must filter by location, not show a global list.

## Voucher lifecycle — observed data (dev 2026-07-08)

- `ACTIVE`: 9,846 rows (~99.5%)
- `USED`: 54 rows (~0.5%)
- `CANCELLED`: 0 rows (declared in code path but no data yet)

9,900 total vouchers in dev — non-trivial volume. Tests must not depend on preexisting fixtures.

## coupon_type_id — TIME-only in dev

`SELECT DISTINCT coupon_type_id FROM company.coupon` → only `1`. Confirms:
- The `coupon_type_id = 1` (TIME) hardcode in the script and the backend plan is safe for MVP.
- If new types appear (`FIXED_PRICE`, `PERCENTAGE`) each would need its own `coupon_type_<name>` table matching this pattern.

## Non-obvious facts confirmed

- **`coupon.uuid` is DB-generated** via `uuid_generate_v4()` default. Don't send from Java on INSERT — read it back if needed.
- **`coupon.created_at` and `coupon_voucher.created_at`** default to `now()`. Don't send.
- **`coupon_type_time.coupon_type_id`** has DB-level default `1`. Even the schema reinforces TIME-only MVP.
- **`uuid-ossp` extension is loaded** in dev (evidenced by the `uuid_generate_v4()` default). But we still generate `scan_code` in Java for consistency with the rest of the codebase.

## Business rules (from Israel — closed 2026-07-08)

### Partner immutability

`coupon.partner_id` is NEVER editable post-creation. This is independent of USED-lock — changing partner would be a different coupon. Enforced in Iteration 3 as `COUPON_PARTNER_IMMUTABLE`.

### USED-lock — full scope

If ANY voucher of a coupon has `status = 'USED'`, the following are blocked at service level:
- `coupon_type_time.time_in_minutes`
- `coupon_type_time.time_price`
- `coupon.valid_from`
- `coupon.valid_until` — including "extending expiry" (safer to loosen later if a business need emerges)
- `coupon_rate_restriction.rate` and `.rate_version` — changing restriction changes the deal

APPEND is ALWAYS allowed regardless of USED — only adds vouchers, never mutates the template. Enforced in Iteration 3 as `COUPON_LOCKED_BY_USED_VOUCHERS`.

## Two operation modes (drives the backend design)

- **CREATE** — new template + N vouchers in `[sequence_min, sequence_max]`. Fails if `coupon_code` already exists at that location (protected by `uq_coupon_coupon_code`).
- **APPEND** — add M more vouchers to an existing template. Continues from `MAX(sequence) + 1`. Updates only `coupon.sequence_max`; does NOT touch pricing/validity/rate restrictions. Race between concurrent APPENDs protected by `uq_coupon_voucher_coupon_sequence` (second one fails with `DuplicateKeyException`).
