---
name: Design System — Technical Patterns & Gotchas
description: Validated technical decisions, known pitfalls, and confirmed approaches for the corepark-ui library
type: feedback
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
---
## Never use Tailwind utilities inside the library

ButtonDirective previously used Tailwind utility classes (`bg-primary`, `text-white`, etc.) requiring consumers to set up a `@theme` block. This was replaced with SCSS + CSS custom properties.

**Why:** Tailwind utility classes require the consumer's Tailwind pipeline to know about the library's design tokens — a hidden dependency that breaks silently.
**How to apply:** All visual styles in the library must use CSS custom properties (`var(--color-primary)`, `var(--radius-md)`, etc.). Never generate Tailwind class strings from library code.

## Always run build:fix-paths after ng-packagr build

ng-packagr copies `src/styles.scss` verbatim to dist. The source uses `lib/tokens/` and `lib/components/` prefixes that don't exist in dist. The `build:fix-paths` npm script (`sed -i ''`) strips them.

**Why:** Without the fix, `styles.scss` in dist has broken `@use` paths and any consumer SCSS compilation fails.
**How to apply:** Never manually edit `dist/corepark-ui/styles.scss` — it's regenerated on every build and then fixed by the script. If paths look wrong after build, check that `build:fix-paths` ran.

## Use rsync for local testing between design-system and frontend-commerce

```bash
rsync -a --delete /Users/israel/Dev/design-system/dist/corepark-ui/ \
  /Users/israel/Dev/frontend-commerce/node_modules/@corepark/corepark-ui/
```

**Why:** Avoids publishing to GitHub Packages (npm.pkg.github.com) on every iteration.
**How to apply:** Suggest this after every design-system build during active development. Do not suggest `npm link` — it has hoisting issues with Angular.

## styles.scss must include tokens — single import contract

Consumers should only need `@use 'corepark-ui/styles'` (or the equivalent angular.json entry). Tokens must be bundled into styles.scss.

**Why:** The "two import" requirement was a footgun — easy to forget tokens, leading to unstyled components (missing CSS variables).
**How to apply:** When adding new token files or component SCSS, always add them to `projects/corepark-ui/src/styles.scss`. Never document a "you also need to import tokens separately" step.

## BEM prefix for library classes: `.cp-btn`, `.cp-*`

All library CSS classes use the `cp-` prefix to avoid collisions with consumer stylesheets.

**Why:** Prevents accidental style collisions in consumer apps.
**How to apply:** When adding new components, always prefix classes with `cp-`. Never use generic class names like `.button`, `.card`, `.tooltip`.

## Never hardcode `rgba(r,g,b,alpha)` for token-based colors — use `color-mix()`

Wrong: `background: rgba(0, 175, 170, 0.1);`
Right: `background: color-mix(in srgb, var(--color-primary), transparent 90%);`

**Why:** Hardcoded rgba values don't respond to CSS custom property overrides. If a consumer overrides `--color-primary`, the rgba values stay hardcoded and break the theming contract.
**How to apply:** Any time you need an alpha variant of a token color, use `color-mix(in srgb, var(--color-*), transparent X%)`. The `X%` is the *transparency* (100 − opacity). Examples:
- 10% opacity → `transparent 90%`
- 12% opacity → `transparent 88%`
- 15% opacity → `transparent 85%` (= `--color-primary-alpha` token)
- 20% opacity → `transparent 80%`
Exception: `rgba(255,255,255,...)` on a solid primary-color background (e.g. progress/goal card) is intentional and can stay — white is not a token.

## BEM state convention: `--{modifier}` vs `is-{state}`

- `cp-badge--success`, `cp-badge--md` — structural variants and sizes (never change at runtime from interaction)
- `is-floating`, `is-focused`, `is-error`, `is-disabled` — interactive/runtime states

**Why:** Consistent with existing component patterns in the library. Makes it easy to distinguish static modifiers from dynamic state in templates and SCSS.
**How to apply:** Use `--` BEM modifier for variant/size inputs. Use `is-` state classes for states driven by user interaction or validation.
