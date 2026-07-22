---
name: Prefer pipes over component methods for pure template transforms
description: When a template needs a pure data transformation (format, mapping, derivation), reach for an Angular pipe instead of a component method. Pipes get memoization and cleaner templates for free.
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Rule:** In Angular templates, use **pipes** for pure transformations. Only use component methods when the transform depends on component state or has side effects.

**Why:** The user asked "por qué no creamos pipes en vez de tener métodos en línea?" (2026-07-09) after I had shipped a rates page with several inline format methods (`rateTypeLabel(id)`, `priceDisplay(rate)`, `formatMinutes(m)`, etc.). Pipes are the idiomatic Angular way for pure template transforms; method calls in templates are a mild code smell because:

1. **Methods re-run on every change-detection cycle**. Pipes with `pure: true` (the default) memoize by reference-equality of the input — the transform runs only when the input actually changes.
2. **Templates read cleaner**: `{{ row | ratePrice }}` beats `{{ priceDisplay(row) }}`. The `data | transform` flow is natural.
3. **Reusable across components without re-importing helper fields** — pipes go into `imports: [...]` once and work.
4. **Component classes stay lean** — no `readonly formatMinutes = formatMinutes` alias lines eating scroll.

**How to apply:**

If a transform is:
- **Pure** (same input → same output, no state, no side effects)
- **Used in a template**
- Not depending on component-instance state

...it should be a pipe.

Structure (following the flat pattern in commerce — see `core/pipes/uuid-pipe.ts`):
```ts
// src/app/core/pipes/thing-pipe.ts
import { Pipe, PipeTransform } from '@angular/core'

@Pipe({ name: 'thing' })
export class ThingPipe implements PipeTransform {
  transform(input: SomeType): string {
    // pure logic
  }
}
```

Import in the component's `imports: [ThingPipe]`. Use in template as `{{ value | thing }}`.

**When methods are still OK:**
- Event handlers (`(click)="onFoo()"`)
- Transforms that depend on component signals/state (though usually a `computed()` is even better)
- Transforms called from other TS code (not just templates)

**How to recognize the refactor opportunity:**
- You've written `readonly fnName = fnName` to expose a util in a component just for template use → probably should be a pipe
- Multiple templates need the same format → definitely a pipe
- Your component class is bloated with formatting helpers → move to pipes and delete the helpers

In the rates migration, this refactor deleted `core/utils/rate-format.ts` entirely and moved all 5 helpers into `core/pipes/*-pipe.ts`.
