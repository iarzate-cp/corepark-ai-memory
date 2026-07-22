---
name: corepark-ui components with no toolbar/header projection slots
description: cp-table only accepts [cpCell] and [cpExpand] as content projection. cp-date-range-picker is intrinsically wide (~600px). Design dialogs around these constraints.
type: reference
originSessionId: e63f3fdb-6cdd-45d9-9e5f-f4a8b4c1ae11
---
Two content-projection facts from `@corepark/corepark-ui@0.0.23` that catch you if you assume Material-style flexibility:

## `cp-table` has no toolbar / header slot

`ngContentSelectors: ["cpCell", "cpExpand"]` — only structural directives, no free-form content. If you need a toolbar with filters/actions next to the table search input, it MUST live outside the `<cp-table>`. Overlay hacks work but are fragile — just put your controls above.

The `filterable` input adds the internal search box; you cannot inject anything next to it.

## `cp-date-range-picker` is intrinsically wide

The inline picker renders two months side-by-side + a preset column (Today, Yesterday, This week…). Effective width ≈ 600px. It does NOT respect container width or offer a compact mode.

Implication: any dialog embedding the shared `<date-range-field>` (which wraps `cp-date-range-picker`) needs to be **≥ 44rem wide** (`704px`) — otherwise the picker's absolute panel triggers horizontal scroll inside the dialog. The Rates Temporary dialog uses `DIALOG_WIDTH_WIDE = '44rem'` for this exact reason (see `project_rates_migration.md`).

Alternative for tight dialogs: portal the picker via CDK Overlay so it escapes the modal boundary — not worth the architectural cost unless the layout demands narrow dialogs. Widening is the pragmatic default.
