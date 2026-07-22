---
name: Early return over ternary in function bodies (TS)
description: User detests ternaries in function/method bodies — always refactor to early returns. Applies to TS code; template ternaries in Angular HTML are still fine (templates lack imperative control flow).
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Rule:** In TypeScript function bodies, **never use ternaries** — even simple one-condition ones. Use early returns.

**Why:** The user said "detesto los ternarios, por qué no usar early return?" (2026-07-09) after I wrote `return m === 0 ? 'a' : 'b'` in a helper. CLAUDE.md already documents "guard first, no if/else" and "no complex ternaries", but he's stricter than that — **any ternary in a function body should be a red flag**, not just complex ones. This reinforces the CLAUDE.md rule as a hard preference.

**How to apply:**

Wrong:
```ts
export const formatMinutes = (m: number): string => {
  if (m < 60) return `${m}m`
  const h = Math.floor(m / 60)
  const min = m % 60
  return min === 0 ? `${h}h` : `${h}h ${min}m`  // ❌ ternary
}
```

Right:
```ts
export const formatMinutes = (m: number): string => {
  if (m < 60) return `${m}m`

  const h = Math.floor(m / 60)
  const min = m % 60
  if (min === 0) return `${h}h`

  return `${h}h ${min}m`
}
```

**Exceptions (still OK):**
- **Template ternaries** in Angular HTML (`{{ x ? a : b }}`, `[class.foo]="active ? 'on' : 'off'"`) — templates don't have `if/return`, ternary is the standard.
- **Nullish coalescing** (`x ?? default`) — not a ternary, always OK.
- **Assignment to const** with a very simple ternary might be tolerable in some contexts, but default to early return + explicit const first.

If you catch yourself writing `... ? ... : ...` inside a `.ts` file (not `.html`), stop and rewrite with `if` guards.
