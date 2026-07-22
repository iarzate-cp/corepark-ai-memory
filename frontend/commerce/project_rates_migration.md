---
name: Rates module — migration from backoffice to commerce
description: /settings/rates ported to commerce. All 5 rate types + sequencing + tax + comps + cash + transaction code done. Delivered 2026-07-13 on feature/rates-module (not yet merged to feature/staging). Full reference doc in ~/Documents/rates-module-migration.md.
type: project
originSessionId: ad4f5cfa-d336-49eb-bbe4-3711000eab64
---
## Status

**All shipped 2026-07-13** on `feature/rates-module` (base: `main`). Not yet merged to `feature/staging`.

- 5 rate types (Fixed, Flat, Variable, Temporary, Overnight) — create/update/delete/detail.
- Rate sequencing (per-row Add/Edit dialog).
- Tax (create/update/remove from header).
- Compensations dialog with empty-state warning.
- Cash payments toggle with dynamic tooltip.
- Shared `<rate-transaction-code>` toggle used by all 5 rate dialogs.
- Duration picker overlay (`<rate-time-picker>`) for variable tier boundaries.
- Whitelist-based `operator-id-body` interceptor for legacy backoffice endpoints.

## Reference document

Full write-up: `~/Documents/rates-module-migration.md` — architecture, backend patterns, dev workflow, gotchas, deferred work.

## Deferred (post-merge)

1. Mobile row actions — BO switches Detail/Edit/Delete to hamburger menu on small screens; commerce shows 3 text buttons.
2. Viewport-adaptive dialog widths (`RatesLayoutState` in BO).
3. `ngx-mask` (nice-to-have for a real HH:mm mask instead of plain text with pattern).

## Design-system library state

Four accumulated changes in `@corepark/corepark-ui` — **built locally and synced** into commerce's `node_modules/@corepark/corepark-ui/`, **not yet committed or published**. Single commit + publish planned once the feature merges.

- `cp-date-range-picker`: `hidePresets` / `hideActions` / `autoApply` inputs for inline usage.
- `cp-form-field`: `width: 100%` on `:host` and `.cp-field-container`.
- `cp-form-field`: subscribes to `NgControl.valueChanges` so the floating label reacts to programmatic `setValue`.
- `cp-form-field`: `:not(.is-error)` on the focused label rule so error color wins over focus.

Iteration workflow while feature is open: edit design-system → `npm run build` → `rm -rf commerce/node_modules/@corepark/corepark-ui && cp -R design-system/dist/corepark-ui commerce/node_modules/@corepark/`.

## Non-obvious backend contract points

These are worth remembering because they're not derivable from reading the commerce code alone:

- Legacy endpoints (`/backoffice/rate/*`, `/backoffice/location/get-partner-summary`, `/backoffice/valet-app-settings/*`, `/backoffice/sequences/*`) reject payloads without `operatorCompanyId` in the body. Header-only scoping doesn't work. Whitelist interceptor handles it.
- Overnight `partnerProcessPayment: true` = "Front desk collects" (not what the field name suggests). Direct mapping — do not negate.
- Variable's last tier MUST send `toParkedTime: null` explicitly; omitting the key breaks the contract.
- Single-tier rate payloads (Fixed/Flat/Temp/Overnight) must NOT include `fromParkedTime`. Send `prices: [{ price }]` only.
- Temporary date format: `"yyyy-MM-dd'T'HH:mm:'00.000000'"` for start, `"HH:mm:'59.999999'"` for end. No offset, fake microseconds, literal seconds. `DateTime.toISO()` is rejected with `BAD_INPUT_FORMAT`.
- `/backoffice/location/get-partner-summary` returns `partnerName`; commerce's `Partner` type uses `name`. `rates-service.getPartners` remaps via `toPartner()`.
