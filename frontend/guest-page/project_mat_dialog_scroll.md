---
name: MatDialog panelClass ↔ _custom-mat-dialog.scss convention
description: Every new MatDialog with a custom panelClass must have a matching rule in src/assets/styles/_custom-mat-dialog.scss or the dialog panel won't scroll (content below viewport becomes unreachable).
type: project
originSessionId: 867d223b-0e43-40dd-afc5-3cd8b73f7e0d
---
When opening a MatDialog with `panelClass: 'foo-dialog'` (e.g. Freedompay, Stripe, Windcave dialogs), the panel does **not scroll by default** — Angular Material's dialog panel has hidden overflow. Any content below the viewport (like a submit button at the bottom) becomes unreachable.

**Fix pattern:** add a rule in `src/assets/styles/_custom-mat-dialog.scss`:
```scss
.foo-dialog {
    overflow-y: auto;
}
```

**Why:** the file is pulled into global styles, so `panelClass` (which is applied to the `.mat-mdc-dialog-panel` element) receives the override.

**How to apply:** whenever adding a new full-height dialog (`minHeight: '100dvh'` pattern) or any dialog whose content can exceed the viewport, add the corresponding `.panel-class-name { overflow-y: auto }` block. Discovered while integrating Stripe (2026-07-14): the Payment Element rendered a tall form (card + wallets + email + phone) that pushed the Pay button off-screen, and without the scroll rule the guest was stuck.

**Related:** `.windcave-payment-dialog` in the same file uses different overrides (transparent background, no shape) because it's a fullscreen popup — the pattern here is per-dialog customization, not a blanket rule.
