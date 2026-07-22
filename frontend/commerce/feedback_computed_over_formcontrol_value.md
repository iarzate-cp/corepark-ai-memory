---
name: computed() reading FormControl.value is NOT reactive — wrap with toSignal
description: Angular signals gotcha. `computed()` that reads `form.controls.x.value` never invalidates on form changes because the FormControl value is not a signal. Wrap `valueChanges` with `toSignal({ initialValue })` and read that instead.
type: feedback
originSessionId: e63f3fdb-6cdd-45d9-9e5f-f4a8b4c1ae11
---
When a `computed()` reads `this.form.controls.xxx.value`, the computed does NOT re-evaluate on form changes — only when a signal it read changes. This leaves the memoized result stale (usually the value at first evaluation, e.g. `null`) and produces silent bugs: guards that always fire, hints/labels that never show, etc.

**Why:** happened in `partner-create-dialog` — `selectedCountry = computed(() => form.controls.countryId.value)` returned `null` even after `countryId.setValue(...)`, which made the submit guard `!country || !geometry` fire "Address is incomplete" and hid the `+52` phone code hint. `phoneCodeHint` and downstream computeds cascaded stale.

**How to apply:** in components that mirror form state into computeds, wrap each control (or the whole `form.valueChanges`) with `toSignal`:

```ts
readonly #countryIdValue = toSignal(this.form.controls.countryId.valueChanges, {
  initialValue: this.form.controls.countryId.value,
})

readonly selectedCountry = computed(() => {
  const id = this.#countryIdValue()
  if (id === null || id === undefined) return null
  return this.countries().find((c) => c.countryId === id) ?? null
})
```

Same trap for cross-field checks like `passwordsMatch` — do the same with `password` and `confirmPassword` (or fall back to a `FormGroup`-level validator). Rule of thumb: if a `computed()` reads FormControl state and doesn't get an update after `setValue/patchValue`, this is the reason.
