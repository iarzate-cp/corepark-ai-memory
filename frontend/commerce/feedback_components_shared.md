---
name: All components live in shared/components/* with barrels
description: Never nest components inside pages/feature/. Every component dir has an index.ts barrel that re-exports with alias-without-Component-suffix
type: feedback
---

**Rule 1 — location:** every Angular component (dialog, presentational, form widget, etc.) lives under `src/app/shared/components/<component-name>/` — never nested inside `pages/<feature>/<sub-component>/`.

**Why:** Israel called this out on 2026-07-01 after I placed `basic-survey-dialog/`, `see-survey-dialog/`, and `survey-preview/` under `pages/settings/survey/`. His reasoning: "qué va a pasar si necesitamos usar ese componente en otra parte del proyecto, por qué tendría que acceder a la carpeta `pages/settings/survey/...`?" — nesting under a feature creates future reversed-dependency smells when other features (or shared) need the same UI. Even a component that feels "single-use today" should live in `shared/components/*` from day one; the cost is zero and the option value is real.

**Rule 2 — barrels:** every component dir has an `index.ts` that re-exports the class **with the `Component` suffix dropped**, and any exported types.

Example (mirrors `orphan-ticket-dialog/index.ts`):

```ts
export { OrphanTicketDialogComponent as OrphanTicketDialog } from './orphan-ticket-dialog.component'
export type { OrphanTicketDialogData } from './orphan-ticket-dialog.component'
```

**Why:** consumers import from the directory (`import { OrphanTicketDialog } from '@components/orphan-ticket-dialog'`), never reaching into the internal file. If tomorrow the file gets renamed or split, callers don't break. And the aliased short name reads cleaner at call sites (`this.#dialog.open(OrphanTicketDialog, {...})` vs `OrphanTicketDialogComponent`).

**How to apply:**
- New dialog → `shared/components/<dialog-name>/<dialog-name>.component.{ts,html,scss}` + `index.ts`
- New reusable presentational component → same
- Import via `@components/<name>` alias (tsconfig maps to `app/shared/components/*`) — **never** `@components/<name>/<name>.component`
- Pages under `pages/<feature>/` may keep: page components (routed), page-specific templates, and domain helpers (types, pure functions, constants) — but not visual components
- If a shared component needs a domain helper currently under a feature (`pages/<feature>/foo.ts`), move the helper to a shared location too (`@utils/`, `@definitions/`, etc.) — do NOT import from `@pages/*` into shared
