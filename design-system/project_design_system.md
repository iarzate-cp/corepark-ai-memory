---
name: Design System — Architecture & Build Pipeline
description: corepark-ui library structure, build pipeline, dist layout, and how it integrates into frontend-commerce
type: project
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
---
## Repo: `/Users/israel/Dev/design-system`

- **Package name published:** `@corepark/corepark-ui`
- **npm registry:** `https://npm.pkg.github.com` (configured in `.npmrc`)
- **Build tool:** ng-packagr (via `ng build corepark-ui --configuration production`)
- **Build command (full):**
  ```
  ng build corepark-ui --configuration production && npm run build:tokens && npm run build:styles && npm run build:fix-paths
  ```
- **Individual scripts:**
  - `build:tokens` — compiles `projects/corepark-ui/src/lib/tokens/tokens.scss` → `dist/corepark-ui/tokens/tokens.css`
  - `build:styles` — compiles `projects/corepark-ui/src/styles.scss` → `dist/corepark-ui/styles.css`
  - `build:fix-paths` — `sed -i '' 's|lib/tokens/|tokens/|g; s|lib/components/|components/|g' dist/corepark-ui/styles.scss`

## Source structure (`projects/corepark-ui/src/`)

```
lib/
  tokens/           — design tokens (SCSS variables + CSS custom properties)
  components/
    button/         — ButtonDirective (button-directive.ts, button-directive.scss)
    notification/
    ripple/
    select/
    tooltip/
styles.scss         — main consumer entry point (auto-includes tokens + all component styles)
```

## Dist structure (`dist/corepark-ui/`)

```
tokens/             — token SCSS partials (no lib/ prefix — ng-packagr flattens)
components/         — component SCSS (no lib/ prefix)
styles.scss         — corrected by build:fix-paths (tokens/ not lib/tokens/)
styles.css          — compiled CSS version
fesm2022/           — compiled JS bundle (corepark-corepark-ui.mjs)
```

## Critical path-prefix issue

ng-packagr copies `src/styles.scss` verbatim into dist, but the source uses `lib/tokens/` and `lib/components/` prefixes while dist organizes them at `tokens/` and `components/` directly.
**Fix:** `build:fix-paths` runs `sed` after every build to strip the `lib/` prefix in `dist/corepark-ui/styles.scss`.
Without this script the dist `styles.scss` would fail to resolve at compile time for any consumer.

## Subpath imports (tsconfig paths in frontend-commerce)

| Subpath | Contents |
|---|---|
| `corepark-ui/components` | All UI components, directives, services |
| `corepark-ui/tokens` | JS design token constants |
| `corepark-ui/styles` | (SCSS only — not a TS subpath) |

## Local dev sync (no npm publish needed)

```bash
rsync -a --delete /Users/israel/Dev/design-system/dist/corepark-ui/ \
  /Users/israel/Dev/frontend-commerce/node_modules/@corepark/corepark-ui/
```
Run this after every `npm run build` in the design-system repo to reflect changes in frontend-commerce immediately.

**Why:** Avoids publishing a new npm version to GitHub Packages for every iteration during development.
**How to apply:** Suggest this workflow whenever iterating on the design system and testing in frontend-commerce.
