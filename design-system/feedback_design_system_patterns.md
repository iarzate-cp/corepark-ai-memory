---
name: Design System — Technical Patterns & Gotchas
description: Validated technical decisions, known pitfalls, and confirmed approaches for the corepark-ui library
type: feedback
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
modified: 2026-08-27T21:18:20.162Z
---
## Tailwind está eliminado del repo entero (2026-08-27)

Ya no es "no usar Tailwind en la librería" — **no existe en el proyecto**. Se borraron `tailwind.css`, `.postcssrc.json`, las devDeps `tailwindcss`/`@tailwindcss/postcss`/`postcss`, y la entrada de `angular.json`. Solo sobrevive en `node_modules/.pnpm` como *peer opcional* de los builders de Angular y ng-packagr; no es dependencia directa y no pesa en ningún bundle.

**Why:** el CSS de Tailwind vivía **solo en el demo** y nunca entraba en `ng-package.json` ni en `build:styles`. Los componentes de la librería que se estilaban con utilidades enviaban clases **sin CSS** a los consumidores — `cp-radio-*` estaba roto en commerce. Y el `@theme` del demo aliasaba `--color-*`, nombres muertos desde el rename a `--cp-ui-*`, así que el dark mode del demo vía utilidades tampoco funcionaba.
**How to apply:** todo estilo va en `.scss` vía `styleUrl` con BEM `cp-<block>__<element>` + `is-*`, y todo valor sale de un token `--cp-ui-*`. Si hace falta una utilidad compartida, va al lenguaje del demo (`projects/demo/src/styles/_docs.scss`), nunca a la librería. Ver [[project_demo_docs_language]].

## Al quitar Tailwind hay que reponer su preflight

El demo dependía del reset de Tailwind sin declararlo. Vive ahora en `projects/demo/src/styles/_reset.scss`: `box-sizing`, márgenes 0 en headings/p/pre, listas sin bullets, `font: inherit` en controles, `border-collapse`, `display:block` en media.

**Why:** sin él el layout se descuadra en silencio — márgenes UA en cada `<p>` y `<h2>` que antes traían `m-0`.
**How to apply:** cualquier repo que quite Tailwind necesita este paso antes de mirar si "se ve igual".

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

## Los knobs del DS NO se sobreescriben desde un ancestro

Descubierto el **2026-08-31**, tras un intento fallido que parecía correcto.

Cada componente declara sus knobs **en su propio `:host`**:

```scss
:host { --cp-ui-brand-color: var(--cp-ui-color-text-950); }
```

Y **un valor que un elemento declara para sí mismo gana sobre el heredado** — la herencia solo rellena lo que el elemento no define. Así que esto **no hace nada**:

```scss
:host-context([data-design='legacy']) {   /* el host del consumidor */
	--cp-ui-brand-color: var(--color-white);   /* ❌ silencioso */
}
```

Hay que apuntar **al elemento del componente**:

```scss
:host-context([data-design='legacy']) cp-brand {   /* ✅ */
	--cp-ui-brand-color: var(--color-white);
}
```

Funciona porque el elemento vive en la plantilla del consumidor y lleva su atributo de encapsulación: el selector queda en 0,3,1 contra el 0,1,0 del `:host` de la librería.

**Regla:** poner un knob en un ancestro solo sirve si el componente lo deja sin definir. Donde la librería envía un default —o sea, en todos— el selector tiene que casar con el elemento.

**Y ojo con los inputs:** lo que es `input()` y no knob (p. ej. `isotypeColor` de `cp-brand`) CSS no lo alcanza de ninguna forma; hay que bindearlo. `cp-brand` tiene las dos cosas a la vez, así que arreglar solo una deja el lockup a medio pintar.
