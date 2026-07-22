---
name: corepark-ui@0.0.23 API quirks and gotchas
description: Non-obvious details of the design system's public API — table input variance, dialog slots, missing variants, unexported types. Save time by knowing these upfront.
type: reference
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
Design system: `@corepark/corepark-ui@0.0.23`. Types file: `node_modules/@corepark/corepark-ui/types/corepark-corepark-ui.d.ts`. Discovered during Fase 2 of the rates migration (2026-07-09).

## `cp-table` (`CpTableComponent`)

### Inputs demand MUTABLE arrays
`columns: InputSignal<TableColumn[]>` and `rows: InputSignal<unknown[]>` — **not `readonly`**. TypeScript will reject `readonly Rate[]` even though the component only reads them.

**Fix at the boundary**, don't propagate mutability inward:
```ts
readonly rates = this.#state.rates                          // Signal<readonly Rate[]>
readonly rateRows = computed<Rate[]>(() => [...this.rates()]) // mutable copy — pass this to cp-table
```
And declare columns as non-readonly array type: `readonly columns: TableColumn[] = [...]` (the FIELD stays readonly, only the array TYPE PARAMETER loosens).

### Row context in `cpCell` templates is `unknown`
`<ng-template cpCell="key" let-row>` — `row` is typed as `unknown`. Angular's template type-check is LOOSE on `let-*` context variables with unknown/any types (accessing `row.foo` doesn't error). But if the same field is used from a strongly-typed component field, the type check IS strict.

Practical consequence: helper methods called from `cpCell` templates work OK with argument type `Rate`, but if you need `@if (rate.foo === X)` in a component's own template (not `let-row`), TypeScript will complain when `foo` isn't on all variants of a union.

### Collapsible mode
`[collapsible]="true"` + `<ng-template cpExpand let-row>` enables inline row expansion. When rows have action buttons, ALWAYS pair with `[rowClickToggles]="false"` — the whole-row click default conflicts with menus. See `feedback_row_expansion_pattern.md`.

---

## `cp-dialog-content` (`DialogContentComponent`)

### Slots
Three content slots: `*` (default = body), `[extra-action]` (footer left area), `[action]` (footer right area, typically the primary/destructive action).

### `exitLabel` provides the Cancel button automatically
Setting `exitLabel="Cancel"` renders BOTH the header X and a "Cancel" text button in the footer. **Do NOT add a manual Cancel button in the `[action]` slot** — you'll get duplicates. This was a mistake shipped in an early cut of the rate-delete-dialog.

### `[extraActions]="true"` gates the `[extra-action]` slot
Required for the checkbox/gate pattern:
```html
<cp-dialog-content ... [extraActions]="true">
  <cp-checkbox extra-action label="I understand" ... />
  <button action class="danger-btn" ... />
</cp-dialog-content>
```

### `[disabled]="submitting()"` blocks Escape + backdrop click
Use during the actual HTTP request to prevent accidental close mid-flight.

---

## `cpButton` (`ButtonDirective`) — no `danger` variant

`ButtonVariant = 'primary' | 'secondary' | 'text'`. `ButtonSize = 'xs' | 'sm' | 'md'` (default `md`). Height/font by size:

| size | height | font | usage |
|---|---|---|---|
| `md` | 2.625rem (42px) | 1rem | default; page primary CTAs |
| `sm` | 2.125rem (34px) | 0.875rem | dense pages, per-row actions, dialog footers |
| `xs` | 1.75rem (28px) | 0.875rem | very tight table rows |

Also: `[iconOnly]="true"` forces a square button with `aspect-ratio: 1`. The auto-Cancel button inside `cp-dialog-content` is hard-coded to `variant="text" size="sm"` — match your primary action to `size="sm"` to keep the footer visually balanced.

**There is no `danger` variant.** For destructive actions, roll a custom class:

```scss
.danger-btn {
  appearance: none;
  background: var(--color-danger);
  color: #fff;
  border: none;
  border-radius: 0.5rem;
  padding: 0.5rem 1rem;
  font-family: inherit;
  font-size: var(--fs-sm);
  font-weight: var(--fw-medium);
  cursor: pointer;

  &:hover:not(:disabled) {
    background: color-mix(in oklab, var(--color-danger), black 10%);
  }
  &:active:not(:disabled) { transform: scale(0.98); }
  &:disabled { opacity: 0.55; cursor: not-allowed; }
}
```

Use as `<button class="danger-btn">` — plain button, not the `cpButton` directive.

---

## `cp-badge` — variants and export gap

`BadgeVariant = 'success' | 'danger' | 'warning' | 'info' | 'default'` — 5 variants.

**`BadgeVariant` type is NOT exported** from the DS bundle. If you need to type a variable/mapping, redeclare locally:
```ts
type BadgeVariant = 'success' | 'danger' | 'warning' | 'info' | 'default'
```

`[dot]="true"` adds a colored dot for status-like badges.

---

## `NotificationService`

Public API:
- `.show({ type?: 'info'|'success'|'error'|'warning', title: string, description?: string, duration?: number })`
- `.dismiss(id)`
- `.setPosition(position)`

Requires `provideNotifications({ position: 'bottom-right' })` in `app.config.ts` (already wired in commerce).

For HTTP errors, use `notifyHttpError(notif, err, 'Fallback title')` from `@utils/notify` — it maps `err.error.message` from the LegacyEnvelope shape into the description automatically.

---

## `DialogService`

Public API:
- `.open<R = unknown, D = unknown>(component, config?)` — returns `DialogRef<R>`
- `DialogConfig`: `{ data?, width?, maxWidth?, disableClose? }`
- `DialogRef.close(result?)`, `.beforeClosed()`, `.afterClosed()` (both Observable)
- `DIALOG_DATA` (InjectionToken) and `DIALOG_CONFIG` (InjectionToken) for injection inside dialog components

Pattern for a shared dialog component:
```ts
export interface FooDialogData { readonly foo: Foo }

@Component({ imports: [DialogContentComponent, ...] })
export class FooDialogComponent {
  readonly #data = inject<FooDialogData>(DIALOG_DATA)
  readonly #ref = inject<DialogRef<boolean>>(DialogRef)
  readonly foo = this.#data.foo
}
```

Barrel `index.ts` re-exports with alias: `export { FooDialogComponent as FooDialog } from './foo-dialog.component'` — commerce convention.

---

## `cp-checkbox`

Standard signal-input pattern: `[checked]`, `[disabled]`, `label`. Emits `checkedChange` (aliased from `checked` output). Fits the `[extra-action]` slot for confirmation gating.

---

## Handy exported utility tokens

The DS also exports color/spacing/radius/z-index constants (`COLOR_PRIMARY`, `SPACING_4`, `RADIUS_LG`, `Z_MODAL`, etc.) — occasionally useful if you need the raw pixel/hex values in TS (rare — prefer CSS vars in SCSS).
