---
name: cp-table row expansion needs an explicit chevron affordance, not whole-row click
description: When cp-table is [collapsible]="true" and rows contain action buttons, pass [rowClickToggles]="false" so only the chevron cell toggles expansion
type: feedback
---

For any `cp-table` listing with `[collapsible]="true"` that ALSO has interactive controls in its row cells (action menus, dropdowns, buttons), pass `[rowClickToggles]="false"`. Only the chevron cell (leftmost toggle column) should trigger row expansion.

**Why:** Israel called this out on 2026-07-02 while testing the survey listing: whole-row click was interfering with the Actions/Add-survey dropdown menu — accidentally toggling the row when the user just meant to open the menu. His words: "quiero que sólo cuando se le dé click al botón que tiene la chevron, se active el toggle collapse".

**How to apply:**

- On any `cp-table` with `[collapsible]="true"` AND interactive controls (menu triggers, buttons) inside row cells → set `[rowClickToggles]="false"`.
- The chevron cell already has `cursor: pointer` and keyboard support (`Enter` / `Space`) in `@corepark/corepark-ui@0.0.23+`.
- Pure informational tables (no interactive controls in rows) can keep the default (`true`) — clicking anywhere on the row is a valid affordance in that case.
- The design-system input default remains `true` for backward compatibility. Do not push to change the default — pass the opt-out explicitly per consumer.

**Applied at:** `src/app/pages/settings/survey/survey.component.html` — the survey listing.

**Related backend code / DS docs:** `@corepark/corepark-ui@0.0.23` `CpTableComponent` — see the design-system workflow memory in ms-backoffice-service scope (`reference_design_system_workflow.md`) for how to publish DS changes.
