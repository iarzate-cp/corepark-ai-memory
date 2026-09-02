---
name: tokens-muertos-color-sin-prefijo-cp-ui
description: "lib/tokens/index.ts y 2 componentes siguen apuntando a --color-* que no existe desde el rename a --cp-ui-*; bug pendiente, no tocado"
metadata: 
  node_type: memory
  type: project
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T21:38:22.735Z
---

> ✅ **RESUELTO el 2026-08-27.** `lib/tokens/index.ts` se reescribió completo verificado contra los `.scss`, y se arreglaron `donut-chart`, `menu-divider` y el spec del isotipo. Cero `var(--color-*)` en el repo. Nota de API: `COLOR_PRIMARY_DARK`/`_LIGHT` pasaron a `COLOR_PRIMARY_DARKER`/`_LIGHTER` para coincidir con los tokens reales — sin riesgo, nadie usaba esas constantes (0 usos en la librería y en los 4 consumidores). Se conserva el registro por el patrón de fallo.

Encontrado el **2026-08-27** barriendo Tailwind.

El rename masivo a `--cp-ui-*` (junio 2026, ver [[project_pending_work]]) dejó referencias huérfanas:

- **`lib/tokens/index.ts` está muerto por completo.** Las ~45 constantes JS (`COLOR_PRIMARY`, `COLOR_BG_50`, `COLOR_TEXT_950`…) devuelven `'var(--color-primary)'` etc. Es el subpath público **`corepark-ui/tokens`**, documentado en CLAUDE.md — cualquier consumidor que las use obtiene un valor inválido.
- **`components/charts/donut-chart/donut-chart-component.ts`** líneas 6–11: la paleta por defecto son 6 `var(--color-*)` muertos.
- **`components/menu/menu-divider-component.ts`** línea 15: `background: var(--color-text-100)`.
- **`components/logo/corepark-isotype-component.spec.ts`** línea 16 (solo un default de test).

El rename es 1:1 añadiendo el prefijo, **con dos excepciones**:
- `--color-primary-dark` → `--cp-ui-color-primary-darker`
- `--color-primary-light` → `--cp-ui-color-primary-lighter`

Los demás (`-alpha`, `-light`, `-dark` de danger/success/info/warning/yellow/purple) sí existen tal cual bajo `--cp-ui-color-*`.

**Ojo al arreglarlo:** varias constantes de `index.ts` apuntan a nombres que no existen ni con el prefijo nuevo — hay que verificar contra `tokens/_colors.scss` antes del reemplazo masivo, o solo se cambia un set de referencias muertas por otro más bonito.
