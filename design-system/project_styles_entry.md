---
name: styles.scss — Single Consumer Entry Point
description: How corepark-ui/styles.scss was consolidated to auto-include tokens so consumers only need one import
type: project
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
---
## Goal achieved (2026-05-05)

Previously consumers needed two separate imports:
1. `@use 'corepark-ui/tokens'` — to load CSS custom property tokens
2. `@use 'corepark-ui/styles'` — to load component styles

**Fix:** Added `@use 'lib/tokens/tokens'` (source) / `@use 'tokens/tokens'` (dist) at the top of `styles.scss` so a single import covers everything.

## Source file: `projects/corepark-ui/src/styles.scss`

```scss
// ─── CorePark UI — Component Styles ───────────────────────────────────────────
// Import this file once in your app's global stylesheet. Tokens are included automatically.
//
// @use 'corepark-ui/styles';

@use 'lib/tokens/tokens';
@use 'lib/components/button/button-directive';
@use 'lib/components/notification/notification-animations';
@use 'lib/components/ripple/ripple-directive';
@use 'lib/components/select/select-component';
@use 'lib/components/tooltip/tooltip-component';
```

## Dist file: `dist/corepark-ui/styles.scss` (after build:fix-paths)

```scss
@use 'tokens/tokens';
@use 'components/button/button-directive';
@use 'components/notification/notification-animations';
@use 'components/ripple/ripple-directive';
@use 'components/select/select-component';
@use 'components/tooltip/tooltip-component';
```

## Consumer usage (frontend-commerce)

In `angular.json` styles array or `src/styles.scss`:
```scss
@use 'node_modules/@corepark/corepark-ui/styles';
```
Or as a path string in `angular.json`:
```json
"node_modules/@corepark/corepark-ui/styles.scss"
```

This single import provides:
- All CSS custom property design tokens (colors, typography, spacing, radius, shadows, z-index)
- All component styles (button, notification, ripple, select, tooltip)

## frontend-commerce `src/styles.scss` (current state)

After removing the manual tokens workaround:
```scss
@use 'root';
@use 'reset';
@use 'actioner';
@use 'buttons';
@use 'cards';
@use 'inputs';
@use 'alerts';
@use 'dialogs';
@use 'snackbars';
@use 'searcher';
@use 'custom-mat-dialog';
```
The corepark-ui styles are loaded via the `angular.json` styles array entry, not from this file.
