---
name: corepark-ui non-CVA components — binding pattern with Reactive Forms
description: cp-switch, cp-checkbox, and cp-time-picker are NOT ControlValueAccessor in @corepark/corepark-ui@0.0.23. They cannot be plugged into formControlName. Bind via [checked]/(checkedChange) or [value]/(apply) and sync with the form manually. cp-select IS a CVA and works with formControlName.
type: reference
originSessionId: e63f3fdb-6cdd-45d9-9e5f-f4a8b4c1ae11
---
As of `@corepark/corepark-ui@0.0.23`, the following components do NOT implement `ControlValueAccessor`:

- `cp-checkbox` — inputs `checked` (ModelSignal<boolean>), `disabled`, `label`. Output `checkedChange`.
- `cp-switch` — inputs `checked` (ModelSignal<boolean>), `disabled`, `variant`. Output `checkedChange`.
- `cp-time-picker` — inputs `value` (InputSignal<string | null>), `hour12`, `minuteStep`. Outputs `apply`, `cancel`.

**Do NOT do:** `<cp-switch formControlName="x">` — silently no-ops, the FormControl never receives the value.

**Do:** manual binding with two-way pattern.

```html
<cp-switch
  [checked]="!!form.controls.revenueCollected.value"
  (checkedChange)="form.controls.revenueCollected.setValue($event)"
/>

<cp-time-picker
  [value]="form.controls.billTime.value | timeOfDay"
  (apply)="onBillTimeApply($event)"
/>
```

The `cp-time-picker` emits `"HH:mm"` on `apply`; the backend contract for schedules/cut-times in this codebase stores `"HH:mm:ss"` — normalize with `${value}:00` before setting the control. For display, pipe through `timeOfDay` to chop back to `"HH:mm"`.

**`cp-select` IS a ControlValueAccessor** — use `formControlName` directly. Its `options` input takes `SelectOption[]` (`{ value, label, disabled? }`).

**`cp-form-field` is a container**, not a CVA — you compose it with `<input cpInput formControlName="...">` inside. Slots: `[cpPrefix]`, default (the input), `[cpSuffix]`. Inputs: `label`, `error`, `hint`, `variant`, `size`, `disabled`.

Related gotcha: `computed()` that reads FormControl state stays stale unless the reads go through a signal — see `feedback_computed_over_formcontrol_value.md`. If you write `showError()` methods that read `form.controls.x.hasError(...)`, use methods (called every CD) NOT `computed()`.
