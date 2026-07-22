---
name: ButtonDirective — SCSS Migration
description: Full details of the ButtonDirective migration from Tailwind utilities to SCSS with CSS custom properties
type: project
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
---
## What changed (completed 2026-05-05)

`ButtonDirective` was previously generating Tailwind utility class strings (e.g. `bg-primary text-white rounded-lg`) in `hostClass()`. This required consumers to define a `@theme` block in their Tailwind config to expose the design system CSS variables as Tailwind utilities. Without it, buttons rendered with no colors.

**Fix:** Migrated to a SCSS-based `.cp-btn` system using CSS custom properties directly.

## File: `projects/corepark-ui/src/lib/components/button/button-directive.scss`

Full content:

```scss
.cp-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--radius-md);
  font-weight: 700;
  cursor: pointer;
  border: 1px solid transparent;
  white-space: nowrap;
  text-decoration: none;
  transition: background-color 150ms ease, color 150ms ease, border-color 150ms ease;
  line-height: 1;

  &:disabled { cursor: not-allowed; pointer-events: none; }

  &--md { height: 2.625rem; font-size: 1rem; padding-inline: var(--spacing-4); gap: var(--spacing-2); }
  &--sm { height: 2.125rem; font-size: 0.875rem; padding-inline: var(--spacing-3); gap: var(--spacing-2); }
  &--xs { height: 1.75rem; font-size: 0.875rem; padding-inline: var(--spacing-2); gap: var(--spacing-1); }

  &--text:not(&--icon-only) { padding-inline: var(--spacing-2); }
  &--icon-only { padding-inline: var(--spacing-2); padding-block: var(--spacing-2); aspect-ratio: 1; }

  &--primary {
    background-color: var(--color-primary); color: #ffffff; border-color: var(--color-primary);
    &:hover:not(:disabled) { background-color: var(--color-primary-darker); border-color: var(--color-primary-darker); }
    &:active:not(:disabled) { background-color: var(--color-primary-lighter); border-color: var(--color-primary-lighter); }
    &:disabled { background-color: var(--color-bg-100); color: var(--color-text-300); border-color: var(--color-bg-100); }
  }

  &--secondary {
    background-color: transparent; color: var(--color-primary); border-color: var(--color-primary);
    &:hover:not(:disabled) { background-color: var(--color-primary-alpha); }
    &:active:not(:disabled) { background-color: var(--color-primary-subtle); }
    &:disabled { color: var(--color-text-300); border-color: var(--color-text-300); }
  }

  &--text {
    background-color: transparent; color: var(--color-primary); border-color: transparent;
    &:hover:not(:disabled) { color: var(--color-primary-darker); }
    &:active:not(:disabled) { color: var(--color-primary-lighter); }
    &:disabled { color: var(--color-text-300); }
  }
}
```

## File: `projects/corepark-ui/src/lib/components/button/button-directive.ts` — `hostClass()` change

Before (40+ lines of Tailwind string building):
```typescript
protected readonly hostClass = computed(() => {
  // ... 40+ lines of Tailwind utility concatenation
})
```

After (4 lines, semantic BEM classes):
```typescript
protected readonly hostClass = computed(() => {
  const classes = ['cp-btn', `cp-btn--${this.variant()}`, `cp-btn--${this.size()}`]
  if (this.iconOnly()) classes.push('cp-btn--icon-only')
  return classes.join(' ')
})
```

The directive applies classes via: `host: { '[class]': 'hostClass()' }`

## BEM class naming convention

- `.cp-btn` — base
- `.cp-btn--primary` / `--secondary` / `--text` — variants
- `.cp-btn--md` / `--sm` / `--xs` — sizes
- `.cp-btn--icon-only` — modifier for square icon buttons

## Specificity notes

- `&--text:not(&--icon-only)` has specificity 0-2-0 — correctly overrides size `padding-inline` (0-1-0)
- `&--icon-only` declared after size rules — wins by cascade order for `padding-block` and `aspect-ratio`
