---
name: frontend-validation — Token Namespace y Body Background
description: En frontend-validation los tokens --color-bg-* y --color-text-* NO existen; usar siempre --cp-ui-color-* para layouts que hostean componentes de corepark-ui
type: feedback
originSessionId: 4d50cc0f-67f8-4b5c-bcf2-e483eae1dd13
---
En frontend-validation, el namespace propio de tokens (`--color-*`) **no define** tokens de superficie ni de texto compatibles con el design system. Solo tiene `--color-main-*`, `--color-primary-*`, `--color-accent-*`, `--color-grey-*`, `--fs-*`, `--fw-*`.

**El problema**: `_normalizer.scss` pone `body { background: var(--color-grey-500) }` = `hsl(0,0%,13%)` (casi negro). Cualquier layout sin background explícito hereda ese fondo oscuro.

**El fix**: En SCSS de layouts y componentes de frontend-validation que hostean cp-* components, usar los tokens del design system instalado con prefijo `--cp-ui-color-*`:

| Token incorrecto (no existe aquí) | Token correcto (corepark-ui styles.css) |
|---|---|
| `var(--color-bg-app)` | `var(--cp-ui-color-bg-app)` |
| `var(--color-bg-50)` | `var(--cp-ui-color-bg-50)` |
| `var(--color-text-950)` | `var(--cp-ui-color-text-950)` |
| `var(--color-text-700)` | `var(--cp-ui-color-text-700)` |
| `var(--color-text-500)` | `var(--cp-ui-color-text-500)` |
| `var(--color-text-200)` | `var(--cp-ui-color-text-200)` |
| `var(--color-text-100)` | `var(--cp-ui-color-text-100)` |

Los tokens `--cp-ui-color-*` están definidos como light mode en `:root` dentro de `node_modules/@corepark/corepark-ui/styles.css`. Dark mode solo se activa con `[data-theme=dark]` en un ancestro — no hay ningún trigger automático por `prefers-color-scheme` en la librería.

**Why:** El valet layout en frontend-validation se veía oscuro porque usaba `var(--color-bg-app)` (indefinido), heredando el body oscuro de normalizer. Renombrando a `--cp-ui-color-*` se obtuvo el light mode correcto.

**How to apply:** Siempre que escribas SCSS de layout o shell en frontend-validation que necesite colores de superficie/texto del design system, usa `--cp-ui-color-*`, no `--color-*`.
