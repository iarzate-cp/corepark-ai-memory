---
name: In Angular code, prefer RxJS over async/await
description: Israel pushed back on async/await in a component method; align with the codebase's RxJS convention even for one-shot Promise-based browser APIs
type: feedback
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
**Rule:** in Angular code (components, services, states), wrap Promise-based browser APIs in RxJS `from()` and consume via `.pipe(takeUntilDestroyed(...)).subscribe({ next, error })` instead of `async/await`.

**Why:** Israel called this out on 2026-07-01 after seeing `async onCopyUrl(...): Promise<void>` in `frontend-commerce`. The commerce codebase is uniformly RxJS-based. Mixing async/await in one method inside an otherwise Observable-based component creates cognitive friction. Also:
- `takeUntilDestroyed(this.#destroyRef)` gives free cancellation on component destroy — `await` on a Promise keeps running after destroy and can `.set()` a signal on a destroyed component.
- Composable pipelines (`retry`, `switchMap`, `debounceTime`) plug in without refactor.
- CLAUDE.md prescribes RxJS patterns for HTTP; extending the spirit to all async is consistent.

**How to apply:**
- `navigator.clipboard.writeText(x)` → `from(navigator.clipboard.writeText(x)).pipe(takeUntilDestroyed(this.#destroyRef)).subscribe(...)`.
- Any browser API returning a Promise → same treatment.
- HTTP: keep using Angular `HttpClient` (already Observable).
- Applies to `frontend-commerce` and by extension `frontend-backoffice` / `frontend-validation` (same style). `frontend-survey` follows the same pattern (Angular 17 codebase, also Observable-first).
