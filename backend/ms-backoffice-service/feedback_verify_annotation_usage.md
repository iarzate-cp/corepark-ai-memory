---
name: Verify annotation usage before removing or defending it
description: When Lombok annotations (@EqualsAndHashCode, @ToString, etc.) get flagged in review, don't reason from convention — grep the callers. Rule learned from the survey annotations review 2026-07-07
type: feedback
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
When a code reviewer asks *"do we actually use this annotation?"* — never answer from convention or intuition. Always grep the callers and answer with file:line evidence.

**Why:** on 2026-07-07 a reviewer questioned `@EqualsAndHashCode` and `@ToString` on the survey DTOs. My first instinct was to say all of them looked justified. I forced myself to grep instead, and found that 3 of 9 were actually dead weight — the reviewer was right about those. The other 6 were justified by uses in the `RowMapper` that I could point at with file:line.

**How to apply:**

For each annotation flagged, run at least one of:
- `grep -rn 'ClassNameHere' src --include='*.java' | grep -E 'Set<|Map<|contains\(|equals\(|hashCode\('` to find dedup / equality uses.
- Trace the type's role: is it an internal domain object that lives inside collections (needs `equals`/`hashCode`), or a pass-through DTO that only travels from DAO → controller → wire (does not need them)?

Answer template for the reviewer:
- Justified: *"Used at `Path/File.java:LINE` inside `Set<X>` / `Map<X, ?>` — dedup breaks without `equals`/`hashCode` because `Set.contains` falls back to identity comparison."*
- Not justified: *"Confirmed no usage in Set/Map/equals/contains across `src/**/*.java`. Removing."*

**Corollary — don't sprinkle annotations "just in case":** newly created DTOs that don't hit dedup collections should not carry `@EqualsAndHashCode`. Once added, they become residue nobody wants to touch. Better to add on-demand when a real use case appears.

**Corollary — legacy envelope residue:** annotations like `@EqualsAndHashCode(callSuper = false)` on classes that used to `extends Response` (legacy envelope) become orphans after migration to `ApiResponse<T>`. Sweep them out during envelope migrations, don't wait for a reviewer to catch them.
