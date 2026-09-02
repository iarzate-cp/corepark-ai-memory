---
name: cp-app-layout-cp-app-nav-cp-app-nav-footer
description: "API, knobs y comportamiento del shell de app autenticada del DS y su familia de navegación (rail, footer y barra móvil)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-08-31T20:35:17.032Z
---

Extraídos del `app-layout` del **frontend-backoffice** y generalizados. Construidos 2026-08-26; **knobs y comportamiento actualizados el 2026-08-31** con el trabajo de responsive — ver [[project_responsive_shell]] para el razonamiento y los bugs.

## `cp-app-layout` — `lib/layouts/app-layout/`

Chasis puro: rail fijo + área de contenido. Cero estado.

**Un input:** `product` (string) — "BackOffice" / "Partner" / "CRM". Se renderiza bajo el slot de marca. Hoy solo lo usa el BO: `cp-brand` lleva el nombre del proyecto y lo supersede (ver [[project_cp_brand]]).

**Slots:** `[cpAppBrand]`, `[cpAppNav]`, `[cpAppUser]`, default (contenido), `[cpAppMobile]`.

**Único `imports: []`:** `CdkScrollable`, aplicado al `<main>`. **No es adorno** — ver el modelo de scroll abajo.

**Knobs (valores actuales):**
```
--cp-ui-app-layout-aside-width: 15.5rem
--cp-ui-app-layout-aside-bg: var(--cp-ui-color-bg-60)
--cp-ui-app-layout-aside-border: var(--cp-ui-color-text-200)
--cp-ui-app-layout-aside-padding: 0        ← el rail NO tiene padding
--cp-ui-app-layout-brand-padding: 0 var(--cp-ui-spacing-4)
--cp-ui-app-layout-nav-fade: 1.25rem       ← distancia del fade; 0 lo apaga
--cp-ui-app-layout-nav-padding: var(--nav-fade) var(--cp-ui-spacing-4)
--cp-ui-app-layout-footer-padding: 0 var(--cp-ui-spacing-4) var(--cp-ui-spacing-4)
--cp-ui-app-layout-brand-height: 6rem      ← min-height, lockup de 2 líneas
--cp-ui-app-layout-main-padding: 2rem
--cp-ui-app-layout-main-bg: var(--cp-ui-color-bg-app)
--cp-ui-app-layout-mobile-bar-height: 4rem ← lo que reserva el main abajo
--cp-ui-app-layout-height: 100svh          ← ya NO es `auto`
--cp-ui-app-layout-main-overflow: auto
```

**Modelo de padding: por bloque.** El aside no tiene padding; cada bloque (brand/nav/footer) pone el suyo. Los hijos NO añaden inset propio. **Consecuencia práctica:** `cp-app-nav-footer` no trae padding, así que al reutilizarlo fuera del rail (p. ej. en la hoja móvil) hay que darle inset desde el contenedor.

**Estructura:** rail `position: fixed` + `main` con `padding-left: calc(ancho + padding)`. El rail nunca scrollea (`overflow: hidden`); scrollea `.cp-app-layout__nav`, que además lleva el `mask-image` del fade, y el footer va anclado abajo (`margin-top: auto`). El aside es `height: 100svh`.

**Breakpoint del rail: `$tablet` (48rem/768px)** — era `$laptop` hasta el 2026-08-31. **No es un knob**, es un media query: el consumidor no lo puede cambiar, así que cualquier lógica suya de "el rail está visible" tiene que apuntar a 768 (le pasó al BO con tres `1024`).

**Scroll:** el documento no scrollea; el `main` sí, por dentro. Los dos knobs de altura/overflow se mueven juntos. Detalles y consecuencias en [[project_responsive_shell]].

## `cp-app-nav` — `lib/components/app-nav/`

**Inputs:** `routes: readonly AppNavRoute[]` (required), `expanded: boolean`. **Output:** `routeClick`.

```ts
interface AppNavRoute {
  label: string
  route?: string           // opcional: grupos puros sin ruta
  icon?: TablerIcon
  children?: readonly AppNavChild[]
}
```

**Modo colapsable (rail):** acordeón. Solo un grupo abierto (`#openKey`), clave = `route ?? label`, abrir un grupo **navega a su primer hijo**, cerrar el abierto NO navega, y un `effect` sigue la URL para colapsar todo si ninguna ruta con hijos la contiene.

**Modo `expanded` (hoja móvil):** todos los grupos abiertos, sin chevrons.

**Dos cosas que se separaron y no hay que volver a fusionar:**

- `isOpen(route)` → `aria-expanded` y si se renderizan los hijos.
- `isGroupActive(route)` → la clase `is-active`. En `expanded` usa `holdsUrl(url, route)`; en colapsable, `#openKey`. **Atarlos juntos pintaba el menú entero como activo** en la hoja.

**No lleva `overflow`.** El contenedor scrollea; el componente no. Dos scrollers anidados se pelean por la rueda y el mask solo funciona en el que recorta.

**`app-nav-url.ts`** tiene lo compartido: `isWithin`, `holdsUrl` y `currentUrl(router)`. Ese último es un `computed` sobre `router.lastSuccessfulNavigation` — nada de `toSignal(router.events)`, ver [[feedback_signals_over_observables]].

**Knobs:** `--cp-ui-app-nav-gap: 0.35rem`, `-rail`, `-marker`, `-child-height: 1.5rem`, `-child-gap`, `-chevron-size: 0.75rem`.

## `cp-app-nav-bar` — la navegación bajo 768px

**Dos triggers fijos: menú y cuenta.** API completa e invariantes en [[project_responsive_shell]] — en particular: **las dos superficies viven siempre en el DOM**, nunca dentro de un `@if`, porque el bloque de cuenta es contenido proyectado.

## `cp-app-nav-footer`

**Inputs:** `heading`, `name`, `email`, `avatarIcon`. Acciones por **proyección** (no data-driven: una navega, otra abre menú, otra cierra sesión). Gap propio `--cp-ui-app-nav-footer-gap: 0.35rem`. Sin padding propio.

**Las tres apps lo envuelven en un `user-settings` con `:host { display: contents }`** y lo renderizan dos veces: rail y hoja móvil.

## `NavActionDirective` — `[cpNavAction]`

Aplica `.cp-nav-action`, cuyos estilos son **globales** (en `styles.scss`, como `cpButton`) para que funcionen también en contenido proyectado por el consumidor.

- `padding-inline: 0` en reposo → `var(--cp-ui-spacing-4)` en `:hover`, `:focus-visible` y `.is-active`, con transición, para que la etiqueta se deslice dentro de la pastilla
- `padding-block: 0.5rem` (no altura fija)
- `font-weight` regular → semibold en activo
- Iconos a `stroke-width: 1.25` vía CSS (no vía el input `stroke`) para cubrir iconos proyectados
- Radio = `--cp-ui-radius-md` (igual que los botones)

**Depende de que su contenedor aporte el inset.**

## Capas z de la familia

`aside` y `cp-app-nav-bar__tabs` en `--cp-ui-z-sticky` (200) — nunca coexisten. Backdrop de la hoja en `z-overlay`, hoja en `z-modal`, y la barra sube a `z-modal` mientras la hoja está abierta. La cabecera sticky de la hoja necesita `z-index` propio. Razones en [[project_responsive_shell]].

Ver [[project_layouts_category]] y [[project_cp_auth_layout]].
