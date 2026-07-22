---
name: emitEvent false hides programmatic changes from reactive-form-aware widgets
description: Suppressing FormControl.setValue events with { emitEvent: false } silently breaks any downstream widget that subscribes to valueChanges (like cp-form-field's floating label). Prefer equality checks as the loop guard instead.
type: feedback
originSessionId: ad4f5cfa-d336-49eb-bbe4-3711000eab64
---
**Rule**: don't reach for `{ emitEvent: false }` as the default when calling `setValue` / `patchValue` / `reset` on a FormControl. Only use it when you have a concrete recursion to break.

**Why**: several corepark-ui widgets (and pretty much any Angular Material equivalent) subscribe to `NgControl.valueChanges` to keep their internal state in sync — floating labels, subscript hints, error states, etc. If you set a value with `emitEvent: false`, `valueChanges` doesn't fire and those widgets never notice. In `cp-form-field` this manifests as the floating label sitting on top of the newly-set value because `#hasValue` stays false.

**How to apply**:
- Default: call `setValue(x)` without the second argument.
- If you have a legitimate loop concern (e.g., a `valueChanges` subscription that itself calls `setValue`), guard with an equality check inside the subscriber (`if (newValue === current.value) return`). That's cheaper and doesn't blind other subscribers.
- Reserve `{ emitEvent: false }` for cases where equality guarding is impractical (batch-updating dozens of controls where each intermediate emit would slow things down measurably).

Real case: `rate-variable-tiers` had `applyTierConstraints` and `#syncContiguousBounds` both suppressing events. The picker would set a value via `setValue("1:30")`, sync would copy it with `emitEvent: false`, and the destination `cp-form-field`'s floating label stayed low, overlapping the value. Removing `emitEvent: false` fixed the label; the sync's `if (to === next.value) return` continued to prevent recursion.
