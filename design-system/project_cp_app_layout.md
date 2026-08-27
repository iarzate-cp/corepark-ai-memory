---
name: cp-app-layout-cp-app-nav-cp-app-nav-footer
description: "API, knobs y comportamiento del shell de app autenticada del DS y su navegación (extraídos del app-layout del backoffice)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T03:53:34.428Z
---

Extraídos del `app-layout` del **frontend-backoffice** y generalizados. Construidos 2026-08-26.

## `cp-app-layout` — `lib/layouts/app-layout/`

Chasis puro: rail fijo + área de contenido. Cero estado.

**Un input:** `product` (string) — "BackOffice" / "Partner" / "CRM". Se renderiza bajo el slot de marca. Existe porque las tres apps llevan el mismo wordmark de corepark y si no, no sabes en cuál estás.

**Slots:** `[cpAppBrand]`, `[cpAppNav]`, `[cpAppUser]`, default (contenido), `[cpAppMobile]`.

**Knobs** (valores actuales):
```
--cp-ui-app-layout-aside-width: 15.5rem
--cp-ui-app-layout-aside-bg: var(--cp-ui-color-bg-60)
--cp-ui-app-layout-aside-border: var(--cp-ui-color-text-200)
--cp-ui-app-layout-aside-padding: 0        ← el rail NO tiene padding
--cp-ui-app-layout-brand-padding: 0 var(--cp-ui-spacing-4)
--cp-ui-app-layout-nav-padding: 0 var(--cp-ui-spacing-4)
--cp-ui-app-layout-footer-padding: 0 var(--cp-ui-spacing-4) var(--cp-ui-spacing-4)
--cp-ui-app-layout-brand-height: 6rem      ← min-height, lockup de 2 líneas
--cp-ui-app-layout-main-padding: 4rem
--cp-ui-app-layout-main-bg: var(--cp-ui-color-bg-app)
--cp-ui-app-layout-height: auto            ← poner 100svh si el contenido usa %
```

**Modelo de padding: por bloque.** El aside no tiene padding; cada bloque (brand/nav/footer) pone el suyo. Los hijos NO añaden inset propio.

**Detalles estructurales:** rail `position: fixed` + `main` con `padding-left: calc(ancho + padding)`. El rail nunca scrollea (`overflow: hidden`); scrollea `.cp-app-layout__nav` (`flex: 1 1 auto; min-height: 0`) y el footer va anclado abajo (`margin-top: auto`). Breakpoint `$laptop` (64rem/1024px) — el original del backoffice era 1030px.

## `cp-app-nav` — `lib/components/app-nav/`

**Inputs:** `routes: readonly AppNavRoute[]` (required), `expanded: boolean` (móvil: todo abierto, sin chevrons). **Output:** `routeClick`.

```ts
interface AppNavRoute {
  label: string
  route?: string           // opcional: grupos puros sin ruta
  icon?: TablerIcon
  children?: readonly AppNavChild[]
}
```

**Comportamiento (acordeón):**
- Solo un grupo abierto a la vez (`#openKey: string | null`)
- Clave del grupo = `route ?? label`
- Abrir un grupo **navega a su primer hijo**
- Cerrar el grupo abierto NO navega
- Un `effect` sigue la URL: si ninguna ruta con hijos la contiene, `#openKey = null` → todos colapsan y pierden `is-active`

**Knobs:** `--cp-ui-app-nav-gap: 0.35rem`, `-rail`, `-marker`, `-child-height: 1.5rem`, `-child-gap`, `-chevron-size: 0.75rem`.

## `NavActionDirective` — `[cpNavAction]`

Aplica `.cp-nav-action`, cuyos estilos son **globales** (en `styles.scss`, como `cpButton`) para que funcionen también en contenido proyectado por el consumidor.

- `padding-inline: 0` en reposo → `var(--cp-ui-spacing-4)` en `:hover`, `:focus-visible` y `.is-active`, con transición, para que la etiqueta se deslice dentro de la pastilla
- `padding-block: 0.5rem` (no altura fija)
- `font-weight` regular → semibold en activo
- Iconos a `stroke-width: 1.25` vía CSS (no vía el input `stroke`) para cubrir iconos proyectados
- Knobs: `--cp-ui-nav-action-{padding-block,padding? no,radius,color,hover-bg,active-bg,active-color,weight,active-weight,icon-stroke}`
- Radio = `--cp-ui-radius-md` (igual que los botones)

**Depende de que su contenedor aporte el inset** — no trae padding-inline propio.

## `cp-app-nav-footer`

**Inputs:** `heading`, `name`, `email`, `avatarIcon`. Acciones por **proyección** (no data-driven: una navega, otra abre menú, otra cierra sesión). Gap propio `--cp-ui-app-nav-footer-gap: 0.35rem`.

Ver [[project_layouts_category]] y [[project_cp_auth_layout]].
