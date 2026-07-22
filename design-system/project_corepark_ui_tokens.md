---
name: corepark-ui — Token System
description: Complete design token definitions (SCSS CSS custom properties + JS exports) for corepark-ui — all tokens use --cp-ui-* prefix
type: project
originSessionId: f1d14aab-a36f-4dc1-b961-fde3c10a1821
---
All tokens defined in `projects/corepark-ui/src/lib/tokens/`.
`tokens.scss` imports all partials. Dark mode via `[data-theme='dark']`.
JS constants exported from `lib/tokens/index.ts`.

**⚠️ IMPORTANTE:** Todas las CSS custom properties de la librería tienen el prefijo `--cp-ui-` (renombrado en junio 2026 para evitar conflictos con el backoffice). Antes era `--color-*`, `--spacing-*`, etc. — ahora son `--cp-ui-color-*`, `--cp-ui-spacing-*`, etc.

Las variables internas de componente (`--_*`) NO llevan prefijo — siguen siendo privadas.

---

## Colors — `_colors.scss`

### Backgrounds
```
--cp-ui-color-bg-app   light: #f5f6fa   dark: #111111
--cp-ui-color-bg-50    light: #ffffff   dark: #1e1e1e
--cp-ui-color-bg-60    light: #fafafc   dark: #272727
--cp-ui-color-bg-80    light: #f5f6fa   dark: #2f2f2f
--cp-ui-color-bg-100   light: #ebebf0   dark: #383838
```

### Brand / Primary
```
--cp-ui-color-primary          #00afaa
--cp-ui-color-primary-darker   color-mix(in oklch, #00afaa, black 27%)
--cp-ui-color-primary-lighter  color-mix(in oklch, #00afaa, white 45%)
--cp-ui-color-primary-subtle   color-mix(in oklch, #00afaa, white 88%)
--cp-ui-color-primary-alpha    color-mix(in srgb,  #00afaa, transparent 85%)
```
Note: uses `color-mix()` — requires modern browsers (Chrome 111+, Firefox 113+, Safari 16.2+).

### Text / Neutrals
```
--cp-ui-color-text-950  light: #1e2126   dark: #ffffff
--cp-ui-color-text-700  light: #46505e   dark: #d8d8d8
--cp-ui-color-text-500  light: #6b7b8c   dark: #b8b8b8
--cp-ui-color-color-300  light: #b3bcc6   dark: #9a9a9a
--cp-ui-color-text-200  light: #d7dbe0   dark: #4d4d4d
--cp-ui-color-text-100  light: #edeef1   dark: #414141
```

### Status (same light + dark, each has -light / -dark / -alpha variants)
```
--cp-ui-color-danger   #ff6464
--cp-ui-color-success  #10d782
--cp-ui-color-info     #5b8def
--cp-ui-color-warning  #fdac42
--cp-ui-color-yellow   #fddd48
--cp-ui-color-purple   #ac5dd9
```
Variants: `--cp-ui-color-danger-light`, `--cp-ui-color-danger-dark`, `--cp-ui-color-danger-alpha` (etc.)

### JS exports
`COLOR_BG_APP/50/60/80/100`, `COLOR_PRIMARY`, `COLOR_PRIMARY_DARK`, `COLOR_PRIMARY_LIGHT`,
`COLOR_PRIMARY_SUBTLE`, `COLOR_PRIMARY_ALPHA`, `COLOR_TEXT_950/700/500/300/200/100`,
`COLOR_DANGER/SUCCESS/INFO/WARNING/YELLOW/PURPLE` + `-LIGHT/-DARK/-ALPHA` for each status.

---

## Spacing — `_spacing.scss`

Base unit: 4px (0.25rem)

```
--cp-ui-spacing-0   0
--cp-ui-spacing-1   0.25rem   (4px)
--cp-ui-spacing-2   0.5rem    (8px)
--cp-ui-spacing-3   0.75rem   (12px)
--cp-ui-spacing-4   1rem      (16px)
--cp-ui-spacing-5   1.25rem   (20px)
--cp-ui-spacing-6   1.5rem    (24px)
--cp-ui-spacing-8   2rem      (32px)
--cp-ui-spacing-10  2.5rem    (40px)
--cp-ui-spacing-12  3rem      (48px)
--cp-ui-spacing-14  3.5rem    (56px)
--cp-ui-spacing-16  4rem      (64px)
--cp-ui-spacing-18  4.5rem    (72px)
--cp-ui-spacing-20  5rem      (80px)
--cp-ui-spacing-24  6rem      (96px)
```

---

## Typography — `_typography.scss`

```
--cp-ui-font-family       'Roboto Flex', sans-serif
--cp-ui-font-family-mono  'JetBrains Mono', 'Fira Code', monospace
```

Font weights: `--cp-ui-font-weight-light/regular/medium/semibold/bold` → 300/400/500/600/700

Font sizes:
```
--cp-ui-font-size-h1           3.75rem
--cp-ui-font-size-h2           3rem
--cp-ui-font-size-h3           2.25rem
--cp-ui-font-size-subtitle-1   1.875rem
--cp-ui-font-size-subtitle-2   1.5rem
--cp-ui-font-size-body-1       1.25rem
--cp-ui-font-size-body-2       1.125rem
--cp-ui-font-size-button       1rem
--cp-ui-font-size-caption      0.875rem
--cp-ui-font-size-overline     0.75rem
--cp-ui-font-size-paragraph-lg 1.125rem
--cp-ui-font-size-paragraph-md 0.875rem
--cp-ui-font-size-paragraph-sm 0.75rem
```

Letter spacing: `--cp-ui-letter-spacing-tight/-normal/-wide/-wider` → -0.01em / 0em / 0.04em / 0.08em

---

## Border Radius — `_radius.scss`

```
--cp-ui-radius-sm   0.25rem   (4px)
--cp-ui-radius-md   0.5rem    (8px)
--cp-ui-radius-lg   0.75rem   (12px)
--cp-ui-radius-xl   1rem      (16px)
--cp-ui-radius-pill 624.9375rem (≈ 9999px)
--cp-ui-radius-full 50%
```

---

## Shadows — `_shadows.scss`

All use `color-mix` with `#111118`:
```
--cp-ui-shadow-sm   0 1px 3px + 0 1px 2px
--cp-ui-shadow-md   0 4px 6px + 0 2px 4px
--cp-ui-shadow-lg   0 10px 24px + 0 4px 8px
--cp-ui-shadow-xl   0 20px 48px + 0 8px 16px
```

---

## Z-Index — `_z-index.scss`

```
--cp-ui-z-base      0
--cp-ui-z-raised    10   (sticky headers, floating labels)
--cp-ui-z-dropdown  100  (dropdowns, popovers)
--cp-ui-z-sticky    200  (sticky sidebars)
--cp-ui-z-overlay   300  (backdrop overlays)
--cp-ui-z-modal     400  (modals, drawers)
--cp-ui-z-toast     500  (notifications)
```

---

## Grid — `_grid.scss`

SCSS variables (not CSS custom props):
```scss
$breakpoint-tablet:  48rem   (768px)
$breakpoint-desktop: 75rem  (1200px)
```

CSS custom properties (responsive via @media):
```
Mobile  (<768px):  --cp-ui-grid-cols: 4,  --cp-ui-grid-gutter: 1rem,   --cp-ui-grid-margin: 1.5rem
Tablet  (≥768px):  --cp-ui-grid-cols: 12, --cp-ui-grid-gutter: 1.5rem, --cp-ui-grid-margin: 2.5rem
```
