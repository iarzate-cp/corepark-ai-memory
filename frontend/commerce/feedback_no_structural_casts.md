---
name: Never use structural casts like `(x as { field })` to bypass union types
description: When accessing a property that only exists on some variants of a discriminated union, use `in` operator narrowing or type predicates — never structural casts that evade type-safety.
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Rule:** Structural casts like `(rate as { schedule?: Schedule }).schedule` are a code smell. They discard the type-safety of the discriminated union and hide intent.

**Why:** The user flagged this as "horrenda práctica" (2026-07-09) after I used it in `getSchedule` to access an optional field that only exists on some variants of a `Rate` union. The cast works at runtime but tells TypeScript to trust me instead of proving safety — which defeats the point of modeling with a discriminated union in the first place.

**How to apply:** When you need to access a field that only exists on some variants of a union, use one of these type-safe patterns:

**1. `in` operator (best for one-off access)** — TypeScript narrows the union to the variants that declare the property:
```ts
export const getSchedule = (rate: Rate): Schedule | null =>
  'schedule' in rate ? rate.schedule ?? null : null
```

**2. Type predicate (best when the "has X" concept is reused)**:
```ts
const hasSchedule = (rate: Rate): rate is FlatRate | FixedRate | VariableRate =>
  rate.rateTypeId === RateType.Flat
  || rate.rateTypeId === RateType.Fixed
  || rate.rateTypeId === RateType.Variable
```

**3. Exhaustive switch on the discriminator (best when the result depends on the variant)**:
```ts
switch (rate.rateTypeId) {
  case RateType.Flat:
  case RateType.Fixed:
  case RateType.Variable:
    return rate.schedule ?? null
  case RateType.Temporary:
  case RateType.Overnight:
    return null
}
```

**Never write `(x as { field: T }).field` just to shut TypeScript up.** If TypeScript is complaining, it's either telling you something real about your union, or your union is wrong.
