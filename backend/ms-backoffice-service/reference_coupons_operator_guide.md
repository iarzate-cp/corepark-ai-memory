---
name: Coupons Operator Guide (power-user testing document)
description: The HTML operator guide walking a power user through every coupon flow shipped in the 2026-07-16 audit-history refactor. Same format as the Survey Admin Operator Guide (Guide № 001)
type: reference
---

**Path:** `/Users/israel/Documents/coupons-operator-guide-2026-07-16.html`

Standalone HTML document (Fraunces + Manrope + JetBrains Mono, styled after the Survey Admin Operator Guide). Written for power-user testing of the shipped audit-history refactor on `feature/staging`.

## Sections

01. **The listing** — `/settings/coupons` route, filters, row anatomy (single last-action line), refresh behavior.
02. **Employee attribution** — the acting-employee concept, per-dialog picker, `localStorage` prefill, operator-scoped catalog.
03. **Create — New coupon batch** — full form dialog with employee + partner pickers side-by-side.
04. **Add more vouchers** — APPEND flow with employee picker in the same dialog.
05. **Edit — Update an existing batch** — mutable fields, quantity trim/extend semantics, `COUPON-009` when reducing below `used + cancelled`.
06. **Delete — Soft-delete a batch** — confirm checkbox + employee picker, no USED-lock.
07. **Download — Export vouchers to Excel** — small dialog with `Downloaded by` picker.
08. **History — See everything that happened to a batch** — timeline dialog with colored markers per action (green create / blue append / amber edit / red delete / teal download).
09. **Notifications** — success / warning / error toast semantics.
10. **Error codes** — COUPON-001..009 (008 deprecated, kept for compat).
11. **Reporting issues** — details to include when filing.

## When to reference this

- User asks about a coupon flow at the operational (power-user) level — link them here rather than re-explaining.
- Testing coordination: this is the checklist the power user works through.
- Documentation drift check: if a coupon flow changes, this file needs an update too.

## Shareable / distribution

Israel keeps team-facing documents under `~/Documents` per the shareable-docs convention (`feedback_shareable_docs_location.md`). Copy or link this file when sharing with the power user.
